---
title: 网络库设计
date: 2019-06-26 12:17:01
tags:
- iOS
- 网络
---

### 现有的问题

#### 只支持自己的后台请求

目前采用集约的方式处理调用，调用方只需要简单的提供path，param等信息。`NetworkMananger`，会默认添加公参，并将返回数据做格式化。而这些在第三方API中都是不可以行的。

**解决办法** 

1、参数的提供中添加使用哪些默认参数集合。比如java服务会有一些公参，php服务可能有另一些公参，第三方API也有一些，这里提供选择。

2、数据的处理也是固定流程，可以在NetworkType中提供选项来设置。现成的处理方式封装成一个处理对象。可以直接使用该对象，也可以实现协议，自己提供一个处理对象。或者不处理，由调用方拿到原始数据后自行处理

#### 整个流程非常死板，没办法在各个部分进行自定义

上面提到了在处理参数和对返回数据的处理。比较好的情况是，可以分阶段处理，每个阶段的处理定义协议，并根据现有服务器情况提供默认实现，在特殊场景，由调用方自己实现协议处理。

#### 不支持下载，上传只支持`MultipartForm`形式

这部分可以添加，这方面需求较少，并且不影响整体设计。先不考虑

#### 错误处理比较粗糙

目前错误分为三类，网络异常，服务异常，处理异常。

1、没有拿到服务器返回数据的时候，归属到第一类，网络异常，比如连接超时，连接被断开等等

2、拿到返回信息，其中http code非2xx系列，这部分归到服务异常，比如5xx，4xx等等

3、httpcode是200，但是解析后台数据，code非0，这个是与后天约定的，非0就是出错了，算服务异常

4、这些都没问题，但是处理出错了，就算处理异常，处理异常有两种情况，可能是服务端给的格式有问题，也可能是App处理确实有问题。

后面的处理没什么问题，都是很确定的错误，但是第一类错误却不全是网络问题。

比如连接被断开，连接超时，可能是服务器出了问题，导致连接被断开，或者服务响应不及时导致超时，这个时候报网络异常，会让用户很迷惑。需要仔细分类并根据情况进行报错处理。



### 优化考虑的方面

强需求

* 支持标准的http请求，适应第三方Api的使用
* 完整的错误处理机制
* 对每个阶段提供标准化处理以及自定义处理，增加灵活性

弱需求

* 性能统计（URLSessonMetrics iOS10开始支持）
* 做好隔离，便于后续网络库更新带来的大面积修改或者替换底层网络库的麻烦
* 缓存
* 支持下载（断点续传）



### 基础知识收集

#### 完整的接口请求是什么样子的？

#### alamofire做了其中的哪些事情



### 借鉴

#### 猿题库的网络组件

##### 简介

[猿题库](<https://github.com/yuantiku/YTKNetwork>)网络组件代码是开源的，目前6K星

猿题库采用离散模型，提供delegate和block两种回调方式。下面是支持的特性

- 支持按时间缓存网络请求内容
- 支持按版本号缓存网络请求内容
- 支持统一设置服务器和 CDN 的地址
- 支持检查返回 JSON 内容的合法性
- 支持文件的断点续传
- 支持 `block` 和 `delegate` 两种模式的回调方式
- 支持批量的网络请求发送，并统一设置它们的回调（实现在 `YTKBatchRequest` 类中）
- 支持方便地设置有相互依赖的网络请求的发送，例如：发送请求 A，根据请求 A 的结果，选择性的发送请求 B 和 C，再根据 B 和 C 的结果，选择性的发送请求 D。（实现在 `YTKChainRequest` 类中）
- 支持网络请求 URL 的 filter，可以统一为网络请求加上一些参数，或者修改一些路径。
- 定义了一套插件机制，可以很方便地为 YTKNetwork 增加功能。猿题库官方现在提供了一个插件，可以在某些网络请求发起时，在界面上显示“正在加载”的 HUD。

##### 核心源码介绍

`YTBaseRequest`类封装了一个API应该有的全部信息，并且记录整个请求过程中的所有中间数据和最终的返回信息。当有一个API请求时，只需要继承该类，重写 requestURL，requestMethod，requestArgument，然后start即可以发起一个网络请求。可以设置delegate或者直接使用block来处理回调。

该类中有两部内容，请求所需的信息（path，method，argument），以及返回的数据信息

````objective-c
//请求信息
- (NSString *)requestUrl;
- (nullable id)requestArgument;
- (YTKRequestMethod)requestMethod;
...
  
//返回信息
@property (nonatomic, strong, readonly) NSHTTPURLResponse *response;
@property (nonatomic, readonly) NSInteger responseStatusCode;
@property (nonatomic, strong, readonly, nullable) NSDictionary *responseHeaders;
@property (nonatomic, strong, readonly, nullable) NSData *responseData;
@property (nonatomic, strong, readonly, nullable) NSString *responseString;
````

各类回调逻辑

````objective-c
@property (nonatomic, copy, nullable) YTKRequestCompletionBlock successCompletionBlock;
@property (nonatomic, copy, nullable) YTKRequestCompletionBlock failureCompletionBlock;

- (void)requestCompletePreprocessor;
- (void)requestCompleteFilter;
- (void)requestFailedPreprocessor;
- (void)requestFailedFilter;
````

`YTKNetworkAgent`类封装了AFN作为底层网络的访问。作为实际做请求的类，它分析request信息，然后通过AFN发送网络请求，然后根据AFN的回调，来通知request中的回调。

````objective-c
- (void)addRequest:(YTKBaseRequest *)request;
- (void)cancelRequest:(YTKBaseRequest *)request;
- (void)cancelAllRequests;
- (NSString *)buildRequestUrl:(YTKBaseRequest *)request;

//addRequest 会调用此方法，记录下启动的request
- (void)addRequestToRecord:(YTKBaseRequest *)request {
    Lock();
    _requestsRecord[@(request.requestTask.taskIdentifier)] = request;
    Unlock();
}

//处理结果时，会将request取出，将状态回调到request上
- (void)handleRequestResult:(NSURLSessionTask *)task responseObject:(id)responseObject error:(NSError *)error {
    Lock();
    YTKBaseRequest *request = _requestsRecord[@(task.taskIdentifier)];
    Unlock();
  ....
}
````

有上边这两个基础后面就可以方便的实现，批处理，依赖处理

**批处理**可以添加多个request，start后会循环start所有request，并将代理设置为自己，当完成数等于总数就算完成，回调给外部。

````objective-c
//启动，将全部请求都start
- (void)start {
    if (_finishedCount > 0) {
        YTKLog(@"Error! Batch request has already started.");
        return;
    }
    _failedRequest = nil;
    [[YTKBatchRequestAgent sharedAgent] addBatchRequest:self];
    [self toggleAccessoriesWillStartCallBack];
    for (YTKRequest * req in _requestArray) {
        req.delegate = self;
        [req clearCompletionBlock];
        [req start];
    }
}
// 当全部请求结束，批处理就算结束了
- (void)requestFinished:(YTKRequest *)request {
    _finishedCount++;
    if (_finishedCount == _requestArray.count) {
        [self toggleAccessoriesWillStopCallBack];
        if ([_delegate respondsToSelector:@selector(batchRequestFinished:)]) {
            [_delegate batchRequestFinished:self];
        }
        if (_successCompletionBlock) {
            _successCompletionBlock(self);
        }
        [self clearCompletionBlock];
        [self toggleAccessoriesDidStopCallBack];
        [[YTKBatchRequestAgent sharedAgent] removeBatchRequest:self];
    }
}
````



**串型处理**仍然是处理多个request，启动一个，一个完成，再回调中处理下一个，思路很简单。

````objective-c
- (void)start {
    if (_nextRequestIndex > 0) {
        YTKLog(@"Error! Chain request has already started.");
        return;
    }
		// 调用startNextRequest
    if ([_requestArray count] > 0) {
        [self toggleAccessoriesWillStartCallBack];
        [self startNextRequest];
        [[YTKChainRequestAgent sharedAgent] addChainRequest:self];
    } else {
        YTKLog(@"Error! Chain request array is empty.");
    }
}
//移动数组下标，启动下一个请求
- (BOOL)startNextRequest {
    if (_nextRequestIndex < [_requestArray count]) {
        YTKBaseRequest *request = _requestArray[_nextRequestIndex];
        _nextRequestIndex++;
        request.delegate = self;
        [request clearCompletionBlock];
        [request start];
        return YES;
    } else {
        return NO;
    }
}

- (void)requestFinished:(YTKBaseRequest *)request {
    NSUInteger currentRequestIndex = _nextRequestIndex - 1;
    YTKChainCallback callback = _requestCallbackArray[currentRequestIndex];
    callback(self, request);
  	//一个请求结束，调用下一个
    if (![self startNextRequest]) {
        [self toggleAccessoriesWillStopCallBack];
        if ([_delegate respondsToSelector:@selector(chainRequestFinished:)]) {
            [_delegate chainRequestFinished:self];
            [[YTKChainRequestAgent sharedAgent] removeChainRequest:self];
        }
        [self toggleAccessoriesDidStopCallBack];
    }
}
````

**插件**机制是通过`YTKRequestAccessory`来实现。

````objective-c
@protocol YTKRequestAccessory <NSObject>
@optional
- (void)requestWillStart:(id)request;
- (void)requestWillStop:(id)request;
- (void)requestDidStop:(id)request;
@end
//在YTKBaseRequest中添加，会在三个节点通知到这个Accessory，然后在这里可以根据节点状态做一些操作
//比如介绍中说的，在开始时显示一个hud，在结束时关掉
- (void)addAccessory:(id<YTKRequestAccessory>)accessory {
    if (!self.requestAccessories) {
        self.requestAccessories = [NSMutableArray array];
    }
    [self.requestAccessories addObject:accessory];
}
````

#### casa的CTNetworking

搜索了很多的博客，很多博客都是以casa的一篇分析作基础来写，应该是很被认可，这个库是他[文章](https://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html)的实现。

casa文章论述三个方面

1. 使用哪种交互模式来跟业务层做对接？
2. 是否有必要将API返回的数据封装成对象然后再交付给业务层？
3. 使用集约化调用方式还是离散型调用方式去调用API？

并且旗帜鲜明的给出自己的答案，使用delegate而非block，不将数据封装成对象交付，采用离散的方式调用API。上边的连接中有完整的介绍。

##### 核心源码介绍

类似猿题库的网络组件，这里也有类似的结构，`CTApiProxy`负责发送和处理网络请求，底层使用AFN来实现。

````objective-c
typedef void(^CTCallback)(CTURLResponse *response);
@interface CTApiProxy : NSObject
+ (instancetype)sharedInstance;
- (NSNumber *)callApiWithRequest:(NSURLRequest *)request success:(CTCallback)success fail:(CTCallback)fail;
- (void)cancelRequestWithRequestID:(NSNumber *)requestID;
- (void)cancelRequestWithRequestIDList:(NSArray *)requestIDList;
@end
````

提供call和cancel功能。在call中，会将完成回调，交回到CTAPIBaseManager。

CTAPIBaseMananger类似猿题库的YTBaseRequest，用来定义请求信息，接受CTAPIProxy的回调，然后回调给调用方。



### 设计需要考虑的点

casa提出的三个问题是调用方最关心的问题。他给出了他的答案，但是未必适合我们的情况，我们还需要仔细分析这几方面，种种选择的利弊在作出决策。

#### 离散

通常会有个`NetworkManager`，提供一个简单的数据结构，这个结构包含了一个请求的必要信息。调用方构建这个结构，然后通过`NetworkMananger`发出这个请求，通过block的方式来处理。

##### 优点

- 使用方便，无需调用方做太多事情
- 发送请求的地方，就可以处理回调block
- 全局拦截很容易

##### 缺点

- 调用方只需，也只能提供一个请求的简单结构，对后续的操作没有太多操作空间。
- 单个API的拦截处理不容易做到
- block不容易追踪定位，有可能会延长对象的生命周期（对象持有方式，使用不当）

#### 集约

一般一个接口一个对象。定义baseRequest。每个接口继承这个基类，提供自己的相关信息。

##### 优点

- 子类可以重写父类方法，对接口请求的全过程有更多的控制
- 不管是全局的拦截，还是根据API去拦截都很容易
- 可以提供delegate和block两种方式来返回数据

##### 缺点

- 使用麻烦，简单的API也要定义子类
- 一个API一个子类，通常一个业务有很多API，需要大量类来处理
- 需要一些学习成本，要对基类有所了解，更多的自由，意味着更多的操作



#### 数据组织形式

字典？模型？

考虑由调用者传入模型转换方式，自己服务器的模型大部分都有固定的转换逻辑，可以实现好，调用方也可以传入转换逻辑

```swift
enum ModelTransferType {
  case none //可能是做cache，不需要解析，可能是第三方api，或者本身数据可以直接用
  case custom(Transform<T>) //自定义转换逻辑
  case standard //自己服务的模型，有一套标准的解析方式
}
```



#### 决策

##### 离散 or 集约

集约的方式和离散的方式各有优点，希望支持两者，由调用者自行选择。简单的调用，可以使用集约的方式。比较复杂，需要较多自定义的地方，采用离散的方式。

**集约的实现**，这部分仿照moya，提供一个协议，调用方可以用enum来实现这个协议。使用也是类似。

**离散的实现**，实现一个BaseRequest，实现上述协议。内部采用类似猿题库的实现方案。

不管是集约的实现，还是离散的实现，真正实现网络请求的。还是NetworkMananger。

##### block or delegate

离散的实现会提供block和delegate两种方式。集约的实现就只提供block即可。

##### 模型 or 字典

数据处理也由调用方选择。可以使用标准的方式，也可以选择自定义的方式，或者不加处理。





备注：

一般后台都有多个服务，每个服务会有一些公参，header，host等。定义service可以方便的处理多服务的场景。



### 性能优化

缓存，DNS，keep-alive以及多路复用

### 安全

#### 设计签名

签名可以确保是自己的客户端，而非爬虫。

#### 安全的数据传输

使用https