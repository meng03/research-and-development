---
title: iOS内存介绍
date: 2019-03-23 13:00:23
tags:
- iOS
categories: 
- iOS
- 内存
---

## 概览

系统将程序装载进内存之前，会先构建一个进程虚拟空间，进程装载的物理地址会映射到该虚拟地址上，对进程来说，只需要知道这个虚拟空间即可。虚拟空间的低地址部分映射操作系统相关，用来给进程提供堆操作系统的支持。在往上是堆内存，堆内存用来存储创建的对象，下面要讲的主要是这部分。再往上是栈空间，这部分空间由高地址向低地址走。主要用于方法调用。

![](memory.png)

## 对象的内存分配与布局

对象和数组在内存分配上有些相似点，数组是一块连续的区域，内部分为等大小的块。对象也是一块连续的区域，内部是非等大小的块，都算是聚合类型。对象中只有属性会占用内存空间。所以对象的内存分配主要看对象的属性。

NSObject只有一个属性`isa`

````objective-c
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
````

所以一个NSObject对象，只占用一个Class大小的空间，Class是一个objc_class的指针，所以isa所占空间与平台的指针长度有关。对于64位系统来说，一个NSObject对象，占用8个字节。

一个类，继承自NSObject，并增加一个指针类型的属性，则占用空间增加至16个字节。若是增加一个一字节的char类型，按照这个逻辑空间占用应该是17，但是实际上，这个对象的内存占用是24。因为对象内的布局，并不是属性的无缝连接。为了方便CPU对数据的读写更有效率，对象内部的布局有几个原则，关于原则的由来，可以看看[介绍](https://www.zhihu.com/question/27862634)

内存对齐原则：

- 对于结构体的各个成员，第一个成员的偏移量是0，排列在后面的成员其当前偏移量必须是当前成员类型的整数倍
- 结构体内所有数据成员各自内存对齐后，结构体本身还要进行一次内存对齐，保证整个结构体占用内存大小是结构体内最大数据成员的最小整数倍
- 如程序中有#pragma pack(n)预编译指令，则所有成员对齐以n字节为准(即偏移量是n的整数倍)，不再考虑当前类型以及最大结构体内类型

下面可以看个例子，印证一下

````c++
struct StructOne {
	char a;         //1字节
	double b;       //8字节
	int c;          //4字节
	short d;        //2字节
} MyStruct1;

struct StructTwo {
	double b;       //8字节
	char a;         //1字节
	short d;        //2字节
	int c;         //4字节
} MyStruct2;
NSLog(@"%lu---%lu--", sizeof(MyStruct1), sizeof(MyStruct2));
//24,16
````

## 内存管理

### 管理策略

iOS 采用的内存管理策略为引用计数，对一个对象，每多一个持有者，引用计数+1，每少一个持有者，引用计数减一，当引用计数为0时，也就是没有持有者，就会触发该对象的销毁操作。

### 数据结构

对某个对象的引用计数+1，可以使用retain方法，-1则是release方法，可以用retainCount查看对象的引用数。下面先看下为了实现这些操作所使用的数据结构

引用计数涉及到的数据结构有isa指针，sidetables（sidetable，weaktable）等。

目前iOS设备普遍是arm64系统，指针位数为64位，可以寻址2的64次方的内存，目前完全用不到。所以运行时库对isa指针进行了特殊的处理，下面是非指针类型的isa。只有shiftcls代表的33位是表示指针。

这里引用计数相关的有`has_sidetable_rc`和`extra_rc`，`extra_rc`是用来存储该对象被引用的次数的。但是因为位数有限，能表达的数量很有限，当不够表达时，`has_sidetable_rc`标记是否存储在sidetable中。

````c++
struct {
    uintptr_t indexed           : 1;
    uintptr_t has_assoc         : 1;
    uintptr_t has_cxx_dtor      : 1;
    uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
    uintptr_t magic             : 6;
    uintptr_t weakly_referenced : 1;
    uintptr_t deallocating      : 1;
    uintptr_t has_sidetable_rc  : 1;
    uintptr_t extra_rc          : 19;
};
````

sidetables由64个sidetable组成。通过hash算法定位。因为这些表是全局共享，会频繁的并发读写，如果只有一个表，多个线程同时操作时，要等很久。分表后可以大大减少多个线程同时操作一个表的情况，提高性能。

```c++
struct SideTable {
    // 保证原子操作的自旋锁
    spinlock_t slock;
    // 引用计数的 hash 表
    RefcountMap refcnts;
    // weak 引用全局 hash 表
    weak_table_t weak_table;
};
```

引用计数表以对象指针为key，以引用计数+两个标记位 为value。因为后两个标记位所以，当引用计数需要加减的时候，是从第三位开始。后面的两个标记位，一个是表示是否正在处于dealloc，一个表示是否有若引用，后面还会看到。

weak表以对象指针为key，以引用地址的数组为value，每增加一个weak引用，就添加到这个数组中。

retain，relase等相关的操作都是针对这些结构的添加修改删除。

### 内存操作

#### retainCount

retainCount比较简单，引用计数存在isa指针中，如果溢出还会存在sidetable中，所以先查询isa指针中的extra_rc位，如果溢出继续取出sidetable中的计数，加在一起。

```c
inline uintptr_t 
objc_object::rootRetainCount()
{
    assert(!UseGC);
    if (isTaggedPointer()) return (uintptr_t)this;

    //锁表
    sidetable_lock();
    isa_t bits = LoadExclusive(&isa.bits);
    if (bits.indexed) {//第一位，用来标记是不是指针型isa
        //不是指针型isa
        //先拿isa中的extra_rc
        uintptr_t rc = 1 + bits.extra_rc;
        //如果是放在sidetables中的，就去sidetables中取
        if (bits.has_sidetable_rc) {
            //加一起作为最终结果
            rc += sidetable_getExtraRC_nolock();
        }
        //解锁，返回
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    //非isa指针，直接去sidetable中取值
    return sidetable_retainCount();
}

uintptr_t
objc_object::sidetable_retainCount()
{
    SideTable& table = SideTables()[this];

    size_t refcnt_result = 1;
    
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        // this is valid for SIDE_TABLE_RC_PINNED too
        refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;
    }
    table.unlock();
    return refcnt_result;
}
```

#### retain
retain操作会对引用计数加1，有两种情况，非指针型isa，会先操作isa中的rc位，如果溢出，则拷出一般移入sidetable中。指针型isa，直接操作sidetable加一

```c++
//指针型isa，直接操作sidetable
id objc_object::sidetable_retain()
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.indexed);
#endif
    SideTable& table = SideTables()[this];

    if (table.trylock()) {
        size_t& refcntStorage = table.refcnts[this];
        if (! (refcntStorage & SIDE_TABLE_RC_PINNED)) {
            refcntStorage += SIDE_TABLE_RC_ONE;
        }
        table.unlock();
        return (id)this;
    }
    return sidetable_retain_slow(table);
}

//非指针型isa操作
ALWAYS_INLINE id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    assert(!UseGC);
    if (isTaggedPointer()) return (id)this;
    bool sideTableLocked = false;
    //从isa的extra_rc，拷贝到sidetable的标志
    bool transcribeToSideTable = false;
    isa_t oldisa;
    isa_t newisa;
    do {
        transcribeToSideTable = false;
        //取isa中的数据，作为老数据
        oldisa = LoadExclusive(&isa.bits);
        //新数据暂时用老数据占位，接下来操作新数据
        newisa = oldisa;
        if (!newisa.indexed) goto unindexed;
        // don't check newisa.fast_rr; we already called any RR overrides
        if (tryRetain && newisa.deallocating) goto tryfail;
        uintptr_t carry;
        //新数据rc位加一
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++
        //加一后溢出
        if (carry) {
            // newisa.extra_rc++ overflowed
            //是否处理溢出
            if (!handleOverflow) return rootRetain_overflow(tryRetain);
            // Leave half of the retain counts inline and 
            // prepare to copy the other half to the side table.
            
            if (!tryRetain && !sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
        //比较isa和oldisa，相等就将newsisa写入
        //如果不相等，应该是被其他操作改动，就重试，直到成功
    } while (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits));

    if (transcribeToSideTable) {
        // Copy the other half of the retain counts to the side table.
        sidetable_addExtraRC_nolock(RC_HALF);
    }

    if (!tryRetain && sideTableLocked) sidetable_unlock();
    return (id)this;

 tryfail:
    if (!tryRetain && sideTableLocked) sidetable_unlock();
    return nil;

 unindexed:
    if (!tryRetain && sideTableLocked) sidetable_unlock();
    if (tryRetain) return sidetable_tryRetain() ? (id)this : nil;
    else return sidetable_retain();
}
```



#### release

自动引用减一，减到0，会调用SEL_dealloc，触发dealloc。

```c++
uintptr_t 
objc_object::sidetable_release(bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.indexed);
#endif
    SideTable& table = SideTables()[this];

    bool do_dealloc = false;

    if (table.trylock()) {
        RefcountMap::iterator it = table.refcnts.find(this);
        if (it == table.refcnts.end()) {
            //找不到这个值，就销毁对象
            do_dealloc = true;
            //给这个值设置为，正在销毁标志
            table.refcnts[this] = SIDE_TABLE_DEALLOCATING;
        } else if (it->second < SIDE_TABLE_DEALLOCATING) {
            // SIDE_TABLE_WEAKLY_REFERENCED may be set. Don't change it.
            //引用计数已经减到0了，deallocating还没设置
            //进行销毁，并且，将销毁标志位置1
            do_dealloc = true;
            it->second |= SIDE_TABLE_DEALLOCATING;
        } else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
            //引用计数减一
            it->second -= SIDE_TABLE_RC_ONE;
        }
        table.unlock();
        if (do_dealloc  &&  performDealloc) {
            ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
        }
        return do_dealloc;
    }

    return sidetable_release_slow(table, performDealloc);
}
```

dealloc 注释有说明，如果没有额外处理，就直接free，不然先通过object_dispose处理若引用，关联属性等

```C++
inline void
objc_object::rootDealloc()
{
    assert(!UseGC);
    if (isTaggedPointer()) return;

    //指针型isa && 没被若引用&&没有关联属性&&没有c++创建&&没有用引用计数表
    if (isa.indexed  &&  
        !isa.weakly_referenced  &&  
        !isa.has_assoc  &&  
        !isa.has_cxx_dtor  &&  
        !isa.has_sidetable_rc)
    {
        assert(!sidetable_present());
        //直接释放
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}

```

dispose会调用destructInstance，这个方法如下，注释有说明

```c++
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = !UseGC && obj->hasAssociatedObjects();
        bool dealloc = !UseGC;

        // This order is important.
        //调用c++的销毁方法
        if (cxx) object_cxxDestruct(obj);
        //移除关联属性
        if (assoc) _object_remove_assocations(obj);
        //清理引用计数表和weak表
        if (dealloc) obj->clearDeallocating();
    }

    return obj;
}
```

下面是clearDeallocating，主要是处理引用计数表和弱引用表

```c++
inline void 
objc_object::clearDeallocating()
{
    if (!isa.indexed) {
        // Slow path for raw pointer isa.
        //清理引用计数表
        sidetable_clearDeallocating();
    }
    else if (isa.weakly_referenced  ||  isa.has_sidetable_rc) {
        // Slow path for non-pointer isa with weak refs and/or side table data.
        clearDeallocating_slow();
    }

    assert(!sidetable_present());
}
```

### autoreleasepool

#### 大体思路

autoreleasepool通过`AutoreleasePoolPage`管理对象，每个线程有一个page，存储在TLS中。page内部以栈的方式组织，大小为操作系统的一页，大部分为4096KB。每次添加对象就通过page的压栈存入，存满会创建一个新的page继续存储，page与page通过双向链表的方式存储。push操作会将一个哨兵（nil）压栈，并返回这个位置的地址。pop会查找这个地址，将栈顶到该位置的对象都release。多次的autoreleasepool，会有多个push，记录多个哨兵的位置，然后pop时pop到对应的位置。

#### autoreleasepool的实现

我们通常使用自动释放池就是使用`@autoreleasepool{}`，这个block对应一个结构体

```c++
struct __AtAutoreleasePool {
__AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
void * atautoreleasepoolobj;
};
```

这个结构题会在初始化的时候调用`objc_autoreleasePoolPush`，在析构时调用`objc_autoreleasePoolPop `。
我们在objc源码中找到这两个方法。

```c++
void *
objc_autoreleasePoolPush(void)
{
if (UseGC) return nil;
return AutoreleasePoolPage::push();
}

void
objc_autoreleasePoolPop(void *ctxt)
{
if (UseGC) return;
AutoreleasePoolPage::pop(ctxt);
}
```

这两个方法是`AutoreleasePoolPage`这个类来实现的。
直接从这两个方法看起

```c++
//添加哨兵POOL_SENTINEL（值为nil），处理page，返回哨兵对象的地址。
static inline void *push() 
{
id *dest;
if (DebugPoolAllocation) {
// Each autorelease pool starts on a new pool page.
dest = autoreleaseNewPage(POOL_SENTINEL);
} else {
dest = autoreleaseFast(POOL_SENTINEL);
}
assert(*dest == POOL_SENTINEL);
return dest;
}
```

下面看下怎么处理的poolpage

```c++
static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
```

拿到当前page，如果能拿到并且，page没有存满，就将obj存入
如果page是满的，就走autoreleaseFullPage
如果没拿到page，走autoreleaseNoPage方法

接着看下hotPage()是怎么处理的。

```c++
static inline AutoreleasePoolPage *hotPage() 
{
    AutoreleasePoolPage *result = (AutoreleasePoolPage *)
        tls_get_direct(key);
    if (result) result->fastcheck();
    return result;
}
```

这个page是放到线程的存储空间的，所以poolpage是线程相关的，一个线程，一个page链。

没有page时，第一次创建成功会将hotpage存起来，会存到线程中。

```c++
static inline void setHotPage(AutoreleasePoolPage *page) 
{
if (page) page->fastcheck();
tls_set_direct(key, (void *)page);
}
```

至此push就差不多了，总结下push都干了什么

- 从线程的存储空间中拿到当前页（hotpage），没有的话，就创建一个放进去
- 查看page有没有满，没满就将传入的哨兵存入。
- page满了，向链中寻找最后一个节点，创建一个新的page，parent设置为这个节点，将这个节点设置为hotpage。

接下来看看pop

```c++
static inline void pop(void *token) 
{
    AutoreleasePoolPage *page;
    id *stop;

    page = pageForPointer(token);
    stop = (id *)token;
    if (DebugPoolAllocation  &&  *stop != POOL_SENTINEL) {
        // This check is not valid with DebugPoolAllocation off
        // after an autorelease with a pool page but no pool in place.
        _objc_fatal("invalid or prematurely-freed autorelease pool %p; ", 
                    token);
    }

    if (PrintPoolHiwat) printHiwat();

    page->releaseUntil(stop);

    // memory: delete empty children
    if (DebugPoolAllocation  &&  page->empty()) {
        // special case: delete everything during page-per-pool debugging
        AutoreleasePoolPage *parent = page->parent;
        page->kill();
        setHotPage(parent);
    } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
        // special case: delete everything for pop(top) 
        // when debugging missing autorelease pools
        page->kill();
        setHotPage(nil);
    } 
    else if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```

pop操作和push是成对操作，push操作记录的位置，接下来会用来pop。

#### runtime对autorelease返回值的优化

**问题1:为什么要做这个优化？**

答：
当返回值被返回之后，紧接着就需要被 retain 的时候，没有必要进行 autorelease + retain，直接什么都不要做就好了。

**问题2:如何做的优化？**

基本思路：

在返回值身上调用`objc_autoreleaseReturnValue`方法时，runtime在TLS中做一个标记，然后直接返回这个object（不调用autorelease）；同时，在外部接收这个返回值的objc_retainAutoreleasedReturnValue里，发现TLS中有标记，那么直接返回这个object（不调用retain）。
于是乎，调用方和被调方利用TLS做中转，很有默契的免去了对返回值的内存管理。

具体做法：
优化主要是通过两个方法进行实现`objc_autoreleaseReturnValue`和`objc_retainAutoreleasedReturnValue`

看第一个方法前，先看个枚举

```c++
enum ReturnDisposition : bool {
ReturnAtPlus0 = false, ReturnAtPlus1 = true
};
```

**`objc_autoreleaseReturnValue`**

方法的实现如下，通过注释进行了解释

```c++
// Prepare a value at +1 for return through a +0 autoreleasing convention.
id objc_autoreleaseReturnValue(id obj)
{
//可被优化的场景下，直接返回obj
if (prepareOptimizedReturn(ReturnAtPlus1)) return obj;
//否则还是使用autorelease
return objc_autorelease(obj);
}
```

对可优化场景的判断，在`prepareOptimizedReturn`方法中，参数我们根据上边的枚举已经得知
`ReturnAtPlus1`是`true`,看下这个方法的实现，用注释做了说明。

```c++
static ALWAYS_INLINE bool 
prepareOptimizedReturn(ReturnDisposition disposition)
{
//如果调用方符合优化条件，就返回true，表示这次调用可以被优化
if (callerAcceptsOptimizedReturn(__builtin_return_address(0))) {
//设置dispostion，为后续的objc_retainAutoreleasedReturnValue准备
if (disposition) setReturnDisposition(disposition);
return true;
}
//不符合条件，不做优化
return false;
}
```

`setReturnDisposition`是在TLS中存入标记，后续的`objc_retainAutoreleasedReturnValue`会从TLS中读取来判断，之前是否已经做过了优化。这里比较复杂的方法是`callerAcceptsOptimizedReturn`判断调用方是否接受一个优化的结果。方法的实现比较难以理解，但是注释说明的比较清楚。

> Callee looks for `mov rax, rdi` followed by a call or 
> jump instruction to objc_retainAutoreleasedReturnValue or 
> objc_unsafeClaimAutoreleasedReturnValue. 

接收方为上述的两种情况时，调用方就符合优化条件。这个条件其实是判断，是否MRC和ARC混编，如果调用方和被调方一个使用MRC一个使用ARC，就不能做这个优化了。

**`objc_retainAutoreleasedReturnValue`**

这个方法相对简单，就是判断之前是否已经做了优化（通过TLS中的`RETURN_DISPOSITION_KEY`）

```c++
// Accept a value returned through a +0 autoreleasing convention for use at +1.
id objc_retainAutoreleasedReturnValue(id obj)
{   //从TLS中获取RETURN_DISPOSITION_KEY对应的值，为true，就直接返回obj。
//读取完之后，重置为false
if (acceptOptimizedReturn() == ReturnAtPlus1) return obj;
//没有优化，就走正常的retain流程
return objc_retain(obj);
}
```

这两个方法成对使用，就可以省去将对象添加到autoreleasepool中的操作。

## 应用

### NSMutableArray 的 copy 问题

NSMutableArray copy的返回值是NSArray，但是指针类型是NSMutableArray，所以append操作会报错

```objective-c
NSMutableArray *array = [[NSMutableArray alloc] initWithObjects:@"1",@"2",@"3", nil];
NSMutableArray *a = [array copy];
[a addObject:@"adfa"];

//[__NSArrayI addObject:]: unrecognized selector sent to instance 0x101039810
```

可以将`copy`方法改成`mutableCopy`。

类似的问题，NSMutable**类型的属性，使用`strong`还是`copy`的问题。`copy`会产生不可变对象。操作会报错。

**属性的描述符对get，set方法的影响(MRC)**

**retain**

```objective-c
- (void)setSomeInstance:(SomeClass *)aSomeInstanceValue
{
    if (someInstance == aSomeInstanceValue)
    {
        return;
    }
    SomeClass *oldValue = someInstance;
    someInstance = [aSomeInstanceValue retain];
    [oldValue release];
}
```

这里需要注意的一点时，判等，如果不做这一步，设置同一个对象，release之后，可能



<https://www.cocoawithlove.com/2010/06/assign-retain-copy-pitfalls-in-obj-c.html>

<https://www.cocoawithlove.com/2009/10/memory-and-thread-safe-custom-property.html>

**NSArray类型的property建议用copy而非strong的原因**

如果正常将一个NSArray对象赋值给该属性，那是没有关系的，copy与strong的操作是相同的，但是如果是将一个MutableArray赋值给了该属性，那么用strong属性会有可能出问题，我们对该属性的预期是一个不变的属性，但是原对象发生了改变，是会影响该属性的，copy则会杜绝这一点，对NSMutablaArray的操作，会产生一个不可变数组，符合预期。



### Xcode 工具

#### zombie

为了debug野指针问题（EXC_BAD_ACCESS）。开启zombie模式后，当对象引用计数为0，出发dealloc时，会走特殊的zombie_dealloc。会正常做销毁工作，但是不会释放内存，而是将该isa指针指向一个自定义的zombie_xxx类。在此对这个对象发消息时，会到这个自定义的类。这个类会打印debug。

生成过程

```objective-c
//1、获取到即将deallocted对象所属类（Class）
Class cls = object_getClass(self);

//2、获取类名
const char *clsName = class_getName(cls)

//3、生成僵尸对象类名
const char *zombieClsName = "_NSZombie_" + clsName;

//4、查看是否存在相同的僵尸对象类名，不存在则创建
Class zombieCls = objc_lookUpClass(zombieClsName);
if (!zombieCls) {
//5、获取僵尸对象类 _NSZombie_
Class baseZombieCls = objc_lookUpClass(“_NSZombie_");

//6、创建 zombieClsName 类
zombieCls = objc_duplicateClass(baseZombieCls, zombieClsName, 0);
}
//7、在对象内存未被释放的情况下销毁对象的成员变量及关联引用。
objc_destructInstance(self);

//8、修改对象的 isa 指针，令其指向特殊的僵尸类
objc_setClass(self, zombieCls);
```

触发过程

```objective-c
//1、获取对象class
Class cls = object_getClass(self);

//2、获取对象类名
const char *clsName = class_getName(cls);

//3、检测是否带有前缀_NSZombie_
if (string_has_prefix(clsName, "_NSZombie_")) {
//4、获取被野指针对象类名
  const char *originalClsName = substring_from(clsName, 10);
 
　//5、获取当前调用方法名
　const char *selectorName = sel_getName(_cmd);
　　
　//6、输出日志
　Log(''*** - [%s %s]: message sent to deallocated instance %p", originalClsName, selectorName, self);

　//7、结束进程
　abort();

```



#### address sanitizer

address sanitizer 比 zombie 多了一些功能，除了检查已经被释放了的对象，还可以检查一些越界问题。

{% asset_img ASanVSzombie.png This is an example image %}

##### 实现

开启 address sanitizer 系统的`malloc`和`free`操作会被替换成另一种实现。`malloc`除了会请求指定大小的内存空间，还会在周围申请一些额外空间，并标记为`off-limits`。`free`会将整个区域标记为`off-limits`。然后添加到`quarantine`（隔离，检疫）队列。这个队列会使这块区域延后释放。之后内存的访问也会有些变化。

```c
// 开启前
*address = ...;  // or: ... = *address;

// 开启后
if (IsMarkedAsOffLimits(address)) {
  ReportError(address);
}
*address = ...;  // or: ... = *address;
```

开启设置后，对`off-limits`的访问，会报错。 

##### 性能影响

开启设置后，CPU性能会下降2到5倍，内存使用增加2到3倍。遇到问题后，开启进行复习，还是可以接受的。

##### 局限

1、只能检查执行的代码，必须复现出来。

2、不能坚持内存泄漏，未初始化的内存，以及Int类型的溢出。

#### instruments

instruments关于内存的有 allocation 和 leaks。用于检查内存分配和泄漏，用法比较简单，遇到内存涨幅很大的情况，可以用instrument查看，是那些逻辑导致的，以及查看是否是因为内存泄漏导致的内存只增不减。

### 一些第三方库

因为Xcode提供的工具多是在遇到问题之后才会开启，没办法在开发中实时监测，所以一些开发者写了一些用于在开发期间检查内存问题的库，主要用来检查内存分配和内存泄漏。

#### MLeaksFinder

假定一个controller被pop出去，或者dismiss掉之后，不久就要被释放。hook navigationController的pop相关方法，以及ViewController的dismiss方法，对即将要释放的Controller开启一个定时器，两秒后触发，如果释放了，就结束了。如果没有释放，两秒后就会对这个控制器进行循环引用检查。

延伸一下，当Controller即将释放时，也会调用属性的类似逻辑，让属性触发自检。循环应用检查，是使用的下面要说的库，FBRetainCycle。

#### FBAllocationTracker

这个库主要是追踪分配的对象。有两种模式，1、只追踪分配和释放的数量。2、追踪分配的对象。

首先hook对象的分配，即`allocWithZone`和`dealloc`方法。

第一种模式的实现很简单，在自己的alloc和dealloc中进行加减计数。

第二种模式会额外的将创建的对象加到字典中，dealloc之后移除。这个库在这里比其他类似的库多做了一点操作，就是分代存储。类似instrument中的mark操作，这个库也有一个mark操作，每次mark，就会创建一个新的字典，用于存储这之后的对象，这个字典会放到一个集合中，表示从开启到目前为止的所以分配对象。

#### FBRetainCycle

FBRetainCycleDetector 对可疑对象进行深度优先搜索，查找可能的循环引用，并将循环引用链打印出来。

思路比较简单

- 获取可疑对象的所有strong属性
- 遍历所有属性，对属性继续进行这两步
- 直到没有属性，或者，达到最大深度
- 当有属性是之前遍历过的，说明有循环引用，记录下来

MLeaksFinder找到的只是可疑对象，并不一定是真的泄漏对象。而FBRetainCycle则只能对特定对象进行扫描。两者搭配使用，效果很好。





https://github.com/RetVal/objc-runtime