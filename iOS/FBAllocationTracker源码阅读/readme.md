---
title: FBAllocationTracker 源码阅读
date: 2019-08-01 15:48:54
tags:
- iOS
- 源码
- 内存
---



`FBAllocationTracker`有两种模式，跟踪对象，基数allocs/deallocs。

## 主要类分析

### FBAllocationTrackerManager

本类是提供给使用者的最外层包装。

````objective-c
@interface FBAllocationTrackerManager : NSObject

+ (nullable instancetype)sharedManager;
//启动track
- (void)startTrackingAllocations;
- (void)stopTrackingAllocations;
- (BOOL)isAllocationTrackerEnabled;

- (nullable NSArray<FBAllocationTrackerSummary *> *)currentAllocationSummary;
//使用tracking对象模式
- (void)enableGenerations;
- (void)disableGenerations;
//类似Instruments中allocation提供的mark功能
- (void)markGeneration;

- (nullable NSArray<NSArray<FBAllocationTrackerSummary *> *> *)currentSummaryForGenerations;
- (nullable NSArray *)instancesForClass:(nonnull __unsafe_unretained Class)aCls
                           inGeneration:(NSInteger)generation;
- (nullable NSArray *)instancesOfClasses:(nonnull NSArray *)classes;
- (nullable NSSet<Class> *)trackedClasses;
@end
````

实现中，多是通过`FBAllocationTrackerImpl`来完成。

### FBAllocationTrackerImpl

````c++
AllocationSummary allocationTrackerSummary();
//启动&关闭track
void beginTracking();
void endTracking();
bool isTracking();
//启动&关闭generation
void enableGenerations();
void disableGenerations();
//mark
void markGeneration();

FullGenerationSummary generationSummary();
````

这个类本身实现了allocs和deallocs计数，通过`Generation`类实现track对象功能，后者下面会说到。

与其他内存监控相关库类似，首先需要 hook `alloc` 和 `dealloc`。这部分逻辑大同小异。

**准备 - 将 `allocWithZone`与 `dealloc` 的函数实现，复制到准备好的空方法上**

````c++
void prepareOriginalMethods(void) {
    if (_didCopyOriginalMethods) {
      return;
    }
    // prepareOriginalMethods called from turnOn/Off which is synced by
    // _lock, this is thread-safe
    _didCopyOriginalMethods = true;

      //copy方法的实现到fb_originalAllocWithZone和fb_originalDealloc
      //使用_didCopyOriginalMethods保证只copy一次
    replaceSelectorWithSelector([NSObject class],
                                @selector(fb_originalAllocWithZone:),
                                @selector(allocWithZone:),
                                FBClassMethod);

    replaceSelectorWithSelector([NSObject class],
                                @selector(fb_originalDealloc),
                                sel_registerName("dealloc"),
                                FBInstanceMethod);
  }
````

**开始 - 用自己的alloc和dealloc替换系统的**

````c++
void turnOnTracking(void) {
prepareOriginalMethods();
replaceSelectorWithSelector([NSObject class],
                            @selector(allocWithZone:),
                            @selector(fb_newAllocWithZone:),
                            FBClassMethod);
replaceSelectorWithSelector([NSObject class],
                            sel_registerName("dealloc"),
                            @selector(fb_newDealloc),
                            FBInstanceMethod);
}
````

**关闭 - 用一开始保存的系统方法，再替换回去**

````c++
void turnOffTracking(void) {
    prepareOriginalMethods();

    replaceSelectorWithSelector([NSObject class],
                                @selector(allocWithZone:),
                                @selector(fb_originalAllocWithZone:),
                                FBClassMethod);

    replaceSelectorWithSelector([NSObject class],
                                sel_registerName("dealloc"),
                                @selector(fb_originalDealloc),
                                FBInstanceMethod);
  }
````

自定义的`alloc`和`dealloc`方法除了调用系统实现外，额外调用了记录该类的方法。

````c++
+ (id)fb_newAllocWithZone:(id)zone
{
  id object = [self fb_originalAllocWithZone:zone];
  FB::AllocationTracker::incrementAllocations(object);
  return object;
}

- (void)fb_newDealloc
{
  FB::AllocationTracker::incrementDeallocations(self);
  [self fb_originalDealloc];
}
````

`incrementAllocations`做两件事，将该对象的allocs记数+1。如果启用了`generation`，就将对象记录到`generation`中。`incrementDeallocations`是相反的步骤。

 ````c++
void incrementAllocations(__unsafe_unretained id obj) {
Class aCls = [obj class];

if (!_shouldTrackClass(aCls)) {
  return;
}

std::lock_guard<std::mutex> l(*_lock);

if (_trackingInProgress) {
  (*_allocations)[aCls]++;
}

if (_generationManager) {
  _generationManager->addObject(obj);
}
}
 ````

### Generation

generation主要是用来记录对象。因为可以分代记录。所以generation中有两个集合对象。每一代的对象，记录在同一个`GenerationMap`中。所有的`GenerationMap`都存在`GenerationList`中。`markGeneration`会创建一个新的`GenerationMap`。



## 总结

`FBAllocationTracker`通过对`alloc`和`dealloc`的hook。记录创建和销毁的对象。这部分和其他内存相关的监控类似。此外借鉴了 instruments 工具中 allocation 部分的 markgeneration，提供了分代记录的功能。实现也比较容易，通过集合来存储generation。每次记录对象，就从集合中取最后一个generation来添加即可。