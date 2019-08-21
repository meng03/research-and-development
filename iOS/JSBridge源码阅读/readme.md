#### JSBridge实现的基础
执行js的能力
````objective-c
//WKWebView
- (void)evaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^ _Nullable)(_Nullable id, NSError * _Nullable error))completionHandler;
//UIWebView
- (nullable NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script;
````

拦截URL的能力
````objective-c
//WKWebView
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler;
//UIWebView
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
````
WebViewJavascriptBridge通过这两种能力来实现方便的交互。

#### 实现细节

WebViewJavascriptBridge将native与webview的通信简化成了两个方法
````objective-c
//注册一个处理块
- (void)registerHandler:(NSString*)handlerName handler:(WVJBHandler)handler;
//执行另一端注册的方法
- (void)callHandler:(NSString*)handlerName data:(id)data responseCallback:(WVJBResponseCallback)responseCallback;
````
先看一下native端的注册逻辑
````objective-c
- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler {
    _base.messageHandlers[handlerName] = [handler copy];
}
````
WebViewJavascriptBridge和WKWebViewJavascriptBridge都实现了这个方法，并且实现方式是一致的。就是以handlerName为key，将handler存到base的messageHandlers字典中。如下，JS端也是如此。
````objective-c
//JS端注册handler
function registerHandler(handlerName, handler) {
     messageHandlers[handlerName] = handler;
}
````
注册完，就可以接受调用了，下面看看native的callHandler都做了什么，特别是如何处理回调的。
````objective-c
- (void)callHandler:(NSString *)handlerName data:(id)data responseCallback:(WVJBResponseCallback)responseCallback {
    [_base sendData:data responseCallback:responseCallback handlerName:handlerName];
}
````
WebViewJavascriptBridge和WKWebViewJavascriptBridge都实现了这个方法，并且实现方式是一致的。那我们看看base的sendData做了什么事。
````objective-c
- (void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString*)handlerName {
    //创建一个字典用来将请求参数保存起来
    NSMutableDictionary* message = [NSMutableDictionary dictionary];
    //如果有data，存起来
    if (data) {
        message[@"data"] = data;
    }
    
    //如果有callback，生成一个uniqueId，存起来
    //并且已uniqueId作为key，将callback存到responseCallbacks字典中
    if (responseCallback) {
        NSString* callbackId = [NSString stringWithFormat:@"objc_cb_%ld", ++_uniqueId];
        self.responseCallbacks[callbackId] = [responseCallback copy];
        message[@"callbackId"] = callbackId;
    }
    //如果有handlerName，存起来
    if (handlerName) {
        message[@"handlerName"] = handlerName;
    }
    //添加到消息队列中
    [self _queueMessage:message];
}
````
添加到队列中之后，开始分发消息
````objective-c
- (void)_dispatchMessage:(WVJBMessage*)message {
    //序列化message
    NSString *messageJSON = [self _serializeMessage:message pretty:NO];
    [self _log:@"SEND" json:messageJSON];
    //转义特殊字符
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\\" withString:@"\\\\"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\"" withString:@"\\\""];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\'" withString:@"\\\'"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\n" withString:@"\\n"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\r" withString:@"\\r"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\f" withString:@"\\f"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2028" withString:@"\\u2028"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2029" withString:@"\\u2029"];
    //将message拼接成参数，添加到WebViewJavascriptBridge._handleMessageFromObjC('%@');中
    //最终的js则是执行WebViewJavascriptBridge的_handleMessageFromObjC方法
    NSString* javascriptCommand = [NSString stringWithFormat:@"WebViewJavascriptBridge._handleMessageFromObjC('%@');", messageJSON];
    //主线程执行js
    if ([[NSThread currentThread] isMainThread]) {
        [self _evaluateJavascript:javascriptCommand];

    } else {
        dispatch_sync(dispatch_get_main_queue(), ^{
            [self _evaluateJavascript:javascriptCommand];
        });
    }
}
````
具体执行JS则由最终的容器来执行，UIWebView或者WKWebView。可以看到，这里的callback的实现是类似registerHandler。生成一个uniqueId，添加到responseCallbacks，js执行完对应handler之后会带着结果通过类似callhandler的方式执行这个callback。

接下来看看webview收到消息之后的处理。
之前拼接的js的模式`WebViewJavascriptBridge._handleMessageFromObjC('%@');`很明显，是调用了`WebViewJavascriptBridge的` `_handleMessageFromObjC`，然后将拼接的信息作为参数传递。下面是`WebViewJavascriptBridge的`的`_handleMessageFromObjC`逻辑

````javascript
function _handleMessageFromObjC(messageJSON) {
      _dispatchMessageFromObjC(messageJSON);
}

function _dispatchMessageFromObjC(messageJSON) {
    //解析json
    var message = JSON.parse(messageJSON);
    var messageHandler;
    //如果消息中有回调callbackId
    if (message.callbackId) {
        var callbackResponseId = message.callbackId;
        //创建回调方法
        responseCallback = function(responseData) {
            //
            _doSend({ handlerName:message.handlerName, responseId:callbackResponseId, responseData:responseData });
        };
    }
    //根据handlerName从字典中拿到handler
    var handler = messageHandlers[message.handlerName];
    if (!handler) {
        console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
    } else {
        //执行handler
        handler(message.data, responseCallback);
    }
}

function _doSend(message, responseCallback) {
    if (responseCallback) {
        //拼接callbackId
        var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
        //添加callback到responseCallbacks字典中
        responseCallbacks[callbackId] = responseCallback;
        //callbackId添加到message中
        message['callbackId'] = callbackId;
    }
    //添加message到队列中
    sendMessageQueue.push(message);
    //通过修改src，触发webview的拦截
    messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
}
````



总结来说：
1、前端通过`setupWebViewJavascriptBridge `做一些准备工作，并通过更改src为`https://__bridge_loaded__ `，触发Native端的webview delegate
2、native注入准备好的js，用来创建js端的handlerMap和callbackMap
3、两端各自通过register注册handler
4、两端在需要的时候callhandler来跨平台执行方法。

两端都维护两个map，handlerMap和callbackMap
注册时存储handlerMap
调用时，存储callbackMap，具体逻辑是，生成一个key（cb_自增序列），将callback存入这个key下，然后将key交给另一端。
另一端执行完handler，会通过这个key带着数据调用回来。