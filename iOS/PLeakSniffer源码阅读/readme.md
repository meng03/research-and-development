---
title: PLeakSniffer 源码阅读
date: 2019-07-18 15:48:54
tags:
- iOS
- 源码
- 内存
---



## 介绍

关于PLeakSniffer的介绍可以直接看作者的[文章](<http://mrpeak.cn/blog/leak/>)。下面是原文中的介绍。

> 子对象（比如view）建立一个对controller的weak引用，如果Controller被释放，这个weak引用也随之置为nil。那怎么知道子对象没有被释放呢？用一个单例对象每个一小段时间发出一个ping通知去ping这个子对象，如果子对象还活着就回一个pong通知。所以结论就是：如果子对象的controller已不存在，但还能响应这个ping通知，那么这个对象就是可疑的泄漏对象。



## 使用

介绍比较简单，作者给出的用法如下。

````objective-c
#if MY_DEBUG_ENV
[[PLeakSniffer sharedInstance] installLeakSniffer];
[[PLeakSniffer sharedInstance] addIgnoreList:@[@"MySingletonController"]];
#endif
````

## 实现

### PLeakSnifferCitizen

````objective-c
@protocol PLeakSnifferCitizen <NSObject>
+ (void)prepareForSniffer;
- (BOOL)markAlive;
- (BOOL)isAlive;
@end
````

如果要对某个类型进行内存检查，这个对象要实现这个协议。下面看看库支持的三种类型的实现。

####  NSObject

`NSObject`的分类实现了`markAlive`方法。另外两个方法并没有实现。所以并不是所有继承自`NSObject`的类型都会被追踪。主要是这个方法逻辑覆盖了三种库里支持的类型，所以在这里写，省得每个类型都写一遍。下面看看这个方法的具体逻辑

````objective-c
- (BOOL)markAlive
{
    if ([self pProxy] != nil) {
        return false;
    }
    
    //不处理系统类型
    NSString* className = NSStringFromClass([self class]);
    if ([className hasPrefix:@"_"] || [className hasPrefix:@"UI"] || [className hasPrefix:@"NS"]) {
        return false;
    }
    
    //如果view的superView是nil，则直接返回false
    if ([self isKindOfClass:[UIView class]]) {
        UIView* v = (UIView*)self;
        if (v.superview == nil) {
            return false;
        }
    }
    
    //controller需要在navigation栈中，或者是被present出来，否则返回false
    if ([self isKindOfClass:[UIViewController class]]) {
        UIViewController* c = (UIViewController*)self;
        if (c.navigationController == nil && c.presentingViewController == nil) {
            return false;
        }
    }
    
    //skip some weird system classes
    //跳过一些诡异的系统类（这个好像和上边的UI打头的类型判断重复了）
    static NSMutableDictionary* ignoreList = nil;
    @synchronized (self) {
        if (ignoreList == nil) {
            ignoreList = @{}.mutableCopy;
            NSArray* arr = @[@"UITextFieldLabel", @"UIFieldEditor", @"UITextSelectionView",
                             @"UITableViewCellSelectedBackground", @"UIView", @"UIAlertController"];
            for (NSString* str in arr) {
                ignoreList[str] = @":)";
            }
        }
        if ([ignoreList objectForKey:NSStringFromClass([self class])]) {
            return false;
        }
    }
    
    PObjectProxy* proxy = [PObjectProxy new];
    //给当前对象设置一个代理对象，用于后续的ping操作
    [self setPProxy:proxy];
    [proxy prepareProxy:self];
    
    return true;
}
````

这个方法有点长，因为要分类型处理。主要做两个事情（有点违背方法的单一职责）。

* 如果不符合一些对象的alive定义，则直接返回false，告诉调用方，对象是非alive状态
* 如果对象是alive状态，添加监测代理，用于后续ping对象。

#### UIViewController

`UIViewController`的分类实现了`prepareForSniffer` 和`isAlive`。`markAlive`通过继承`NSObject`获得。

```objective-c
+ (void)prepareForSniffer
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [self swizzleSEL:@selector(presentViewController:animated:completion:) withSEL:@selector(swizzled_presentViewController:animated:completion:)];
        [self swizzleSEL:@selector(viewDidAppear:) withSEL:@selector(swizzled_viewDidAppear:)];
    });
}
```

`prepareForSniffer`做了两个hook。

* hook controller的`present`方法，对`present`出来的controller，调用`markAlive`。
* Hook controller的`viewDidAppear`方法，对属性进行`track`。

第二步，属性的track，其实就是递归调用属性的markAlive，添加proxy。

`isAlive`的判断逻辑

* 所属view在UIWindow视图层级中
* 本身在navigation栈中，或者是由其他controller present出来。

主要代码

````objective-c
UIView* v = self.view;
while (v.superview != nil) {
    v = v.superview;
}
if ([v isKindOfClass:[UIWindow class]]) {
    visibleOnScreen = true;
}
BOOL beingHeld = false;
if (self.navigationController != nil || self.presentingViewController != nil) {
    beingHeld = true;
}
````

#### UINavigationController

`UINavigationController`只实现了`prepareForSniffer`，`isAlive`继承自`UIViewController`。

````objective-c
+ (void)prepareForSniffer
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [self swizzleSEL:@selector(pushViewController:animated:) withSEL:@selector(swizzled_pushViewController:animated:)];
    });
}

- (void)swizzled_pushViewController:(UIViewController *)viewController animated:(BOOL)animated {
    
    [self swizzled_pushViewController:viewController animated:animated];
    
    [viewController markAlive];
    
}
````

Hook push方法，对push的controller，调用markAlive。

#### UIView

`UIView`继承自`NSObject`，还需要实现`prepareForSniffer`和`isAlive`。跟前面几个的实现类似。hook一个合适的时机，调用markAlive。

```objective-c
+ (void)prepareForSniffer
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [self swizzleSEL:@selector(didMoveToSuperview) withSEL:@selector(swizzled_didMoveToSuperview)];
    });
}

- (void)swizzled_didMoveToSuperview
{
    [self swizzled_didMoveToSuperview];
    
    BOOL hasAliveParent = false;
    
    UIResponder* r = self.nextResponder;
    while (r) {
        if ([r pProxy] != nil) {
            hasAliveParent = true;
            break;
        }
        r = r.nextResponder;
    }
    
    if (hasAliveParent) {
        [self markAlive];
    }
}
```

`isAlive`的判断跟之前controller中isAlive的判断前半部分是一样的，通过查看view的最顶层view是不是UIWindow来判断alive。



### PObjectProxy

proxy主要做两件事

* 注册通知接收ping的触发
* 检查宿主是否在不应存活时，还活着，通知出去，此处可能有内存泄漏

#### 注册通知

````objective-c
- (void)prepareProxy:(NSObject*)target {
    self.weakTarget = target;
    [[NSNotificationCenter defaultCenter] removeObserver:self name:Notif_PLeakSniffer_Ping object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(detectSnifferPing) name:Notif_PLeakSniffer_Ping object:nil];
}
````

#### 检查是否泄漏，可能泄漏就post出去

````objective-c
- (void)detectSnifferPing
{
    if (self.weakTarget == nil) {
        return;
    }
    if (_hasNotified) {
        return;
    }
    //如果proxy的target不应该alive。就记录一次fail
    BOOL alive = [self.weakTarget isAlive];
    if (alive == false) {
        _leakCheckFailCount ++;
    }
    //fail次数到达5次以上就会进行提醒，5次大概是2.5秒
    if (_leakCheckFailCount >= kPObjectProxyLeakCheckMaxFailCount) {
        [self notifyPossibleMemoryLeak];
    }
}

- (void)notifyPossibleMemoryLeak
{
    if (_hasNotified) {
        return;
    }
    _hasNotified = true;
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:Notif_PLeakSniffer_Pong object:self.weakTarget];
    });
}
````

### 流程

有了上边的基础设施，下面的流程就比较简单了。流程逻辑主要在`PLeakSniffer`中。使用了一个timer，两个通知。从使用介绍来看，`installLeakSniffer`是起点。

````objective-c
- (void)installLeakSniffer {
    [UINavigationController prepareForSniffer];
    [UIViewController prepareForSniffer];
    [UIView prepareForSniffer];   
    [self startPingTimer];
}
````

三个类型的prepare，以及启动ping定时。三个prepare逻辑上边分析过了，`navigationController`和`UIView`只是在合适的时间`markAlive`。`UIViewControler`除了在present时对presentingController进行标记，还需要对自己的属性进行递归标记。prepare流程结束后，所有之后的controller和View都会纳入监控（通过设置proxy）。

`startPingTimer`会检查是否在主线程，如果不在主线程，dispatch到主线程。然后创建timer，每0.5秒调用一次`sendPing`。`sendPing`就是post一个通知。

````objective-c
- (void)startPingTimer
{
    //检查是否在主线程，非主线程，则dispatch到主线程
    if ([NSThread isMainThread] == false) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self startPingTimer];
            return ;
        });
    }
    
    if (self.pingTimer) {
        return;
    }
    //开启定时，每0.5秒，调用一次sendPing
    self.pingTimer = [NSTimer scheduledTimerWithTimeInterval:kPLeakSnifferPingInterval target:self selector:@selector(sendPing) userInfo:nil repeats:true];
}

- (void)sendPing
{
    //发通知
    [[NSNotificationCenter defaultCenter] postNotificationName:Notif_PLeakSniffer_Ping object:nil];
}
````

前面看过了proxy逻辑，这里sendPing的通知会到proxy中，proxy会检查是否有可能泄漏。如果有泄漏，会通过通知发出去。而接受的地方还是`PLeakSniffer`。接收之后的处理比较简单。如果是需要被忽略的就丢弃。否则alert出来，活着print。

````objective-c
- (void)detectPong:(NSNotification*)notif
{
    NSObject* leakedObject = notif.object;
    NSString* leakedName = NSStringFromClass([leakedObject class]);
    @synchronized (self) {
        if ([_ignoreList containsObject:leakedName]) {
            return;
        }
    }
    
    //we got a leak here
    if (_useAlert) {
        NSString* msg = [NSString stringWithFormat:@"Detect Possible Leak: %@", [leakedObject class]];
        UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"PLeakSniffer" message:msg delegate:nil cancelButtonTitle:nil otherButtonTitles:@"OK", nil];
        [alertView show];
    }
    else
    {
        if ([leakedObject isKindOfClass:[UIViewController class]]) {
            PLeakLog(@"\n\nDetect Possible Controller Leak: %@ \n\n", [leakedObject class]);
        }
        else
        {
            PLeakLog(@"\n\nDetect Possible Leak: %@ \n\n", [leakedObject class]);
        }
    }
}
````

### 补充

前面提到`UIViewController`的`prepareSniffer`hook了`viewDidAppear`方法，对属性进行`track`。这个操作会减少遗漏，最大程度上找到所有可能出现内存问题的地方。但是属性量比较大，不知道这里会不会有内存问题。平常runtime用的少，这里看看别人的实践。

获取所有属性

```objective-c
objc_property_t* properties = class_copyPropertyList(cls, &count );
```

判断属性是否是强引用

````objective-c
bool isStrongProperty(objc_property_t property)
{
    const char* attrs = property_getAttributes( property );
    if (attrs == NULL)
        return false;
    
    const char* p = attrs;
    //property中有'&'则为强引用
    p = strchr(p, '&');
    if (p == NULL) {
        return false;
    }
    else
    {
        return true;
    }
}
````







