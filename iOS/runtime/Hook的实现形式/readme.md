# HOOK的实现形式

## 一个典型的HOOK

````objective-c
				SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        // When swizzling a class method, use the following:
        // Class class = object_getClass((id)self);
        // ...
        // Method originalMethod = class_getClassMethod(class, originalSelector);
        // Method swizzledMethod = class_getClassMethod(class, swizzledSelector);

        BOOL didAddMethod =
            class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
````

这里没有直接使用方法交换`method_exchangeImplementations`交换的原因如下

直接交换 `IMP` 是很危险的。因为如果这个类中没有实现这个方法，`class_getInstanceMethod()` 返回的是某个父类的 `Method`对象，这样 `method_exchangeImplementations()` 就把父类的原始实现（`IMP`）跟这个类的 Swizzle 实现交换了。这样其他父类及其其他子类的方法调用就会出问题，最严重的就是 Crash。

参考：<http://yulingtianxia.com/blog/2017/04/17/Objective-C-Method-Swizzling/>

## FBAllocationTracker中hook的实现

这个库主要是用来跟踪创建的对象，所以对alloc和dealloc进行了hook，这种场景比较简单，非hook任意方法，只是对特定的方法进行hook

先创建空方法，用来保存原方法的实现。

````objective-c
+ (id)fb_originalAllocWithZone:(id)zone{
  return nil;
}

- (void)fb_originalDealloc{}
````

将原方法的实现替换准备好的空方法的实现（保存原方法的实现）

````
replaceSelectorWithSelector([NSObject class],
                                @selector(fb_originalAllocWithZone:),
                                @selector(allocWithZone:),
                                FBClassMethod);

    replaceSelectorWithSelector([NSObject class],
                                @selector(fb_originalDealloc),
                                sel_registerName("dealloc"),
                                FBInstanceMethod);
````

具体实现就是通过replaceMethod来做替换，要区分是实例方法还是类方法

```objective-c
void replaceSelectorWithSelector(Class aCls,
                               SEL selector,
                               SEL replacementSelector,
                               FBMethodType methodType) {

Method replacementSelectorMethod = (methodType == FBClassMethod
                                    ? class_getClassMethod(aCls, replacementSelector)
                                    : class_getInstanceMethod(aCls, replacementSelector));

Class classEntityToEdit = aCls;
//如果要交换类方法，需要获取元类
if (methodType == FBClassMethod) {
  // Get meta-class
  classEntityToEdit = object_getClass(aCls);
}
class_replaceMethod(classEntityToEdit,
                    selector,
                    method_getImplementation(replacementSelectorMethod),
                    method_getTypeEncoding(replacementSelectorMethod));
}
```

当需要开启追踪功能时，就用新的alloc和dealloc替换原实现

```objective-c
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
```

关闭时，就将保存的原实现，拿回来覆盖

````objective-c
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

很健壮，不容易出问题



## Aspects中的实现

Aspects中的hook逻辑稍微复杂一点，不是简单的交换方法。

先将hook信息记录到AspectsContainer中，然后处理class并做hook处理

````objective-c
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    __block AspectIdentifier *identifier = nil;
    aspect_performLocked(^{
        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
            //容器记录了该对象，关于该selector
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
                [aspectContainer addAspect:identifier withOptions:options];

                // Modify the class to allow message interception.
                aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}
````

aspect_hookClass会对class进行处理，构建子类，hook子类的forwardInvocation方法，转到`__ASPECTS_ARE_BEING_CALLED__`中

```objective-c
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {
    NSCParameterAssert(selector);
    Class klass = aspect_hookClass(self, error);
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
  	//记录下selector对象的实现，然后将selector的实现改成_objc_msgForward，来走消息转发流程
    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        SEL aliasSelector = aspect_aliasForSelector(selector);
        if (![klass instancesRespondToSelector:aliasSelector]) {
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
            NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
        }

        // We use forwardInvocation to hook in.
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
        AspectLog(@"Aspects: Installed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
    }
}
```

处理完成后，对该selector的调用就会走到`__ASPECTS_ARE_BEING_CALLED__`

```objective-c
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    NSCParameterAssert(self);
    NSCParameterAssert(invocation);
    SEL originalSelector = invocation.selector;
	SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    invocation.selector = aliasSelector;
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;

    // Before hooks.
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks.
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            //aliasSelector在aspect_prepareClassAndHookSelector方法中，绑定了原方法
            //所以这里是对原方法的调用
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks.
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);

    // If no hooks are installed, call original implementation (usually to throw an exception)
    if (!respondsToAlias) {
        //原方法调用失败，就走原消息转发流程
        invocation.selector = originalSelector;
        SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
        if ([self respondsToSelector:originalForwardInvocationSEL]) {
            ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
        }else {
            [self doesNotRecognizeSelector:invocation.selector];
        }
    }

    // Remove any hooks that are queued for deregistration.
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}
```

此方法会从self的关联属性中取container，然后执行container中添加的hook。



这是一种通过hook forwardInvocation来实现的切面编程，比简单的交换方法实现更加灵活可控



## hook的缺点

[Here are some of the pitfalls of method swizzling](<https://stackoverflow.com/questions/5339276/what-are-the-dangers-of-method-swizzling-in-objective-c>):

- Method swizzling is not atomic
- Changes behavior of un-owned code
- Possible naming conflicts
- Swizzling changes the method's arguments
- The order of swizzles matters
- Difficult to understand (looks recursive)
- Difficult to debug

