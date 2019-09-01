# 探究block的真面目

## 一个极简block的实现

在main中写一个简单的block，用clang，转成C的实现

```objective-c
int main(int argc, const char * argv[]) {
    void (^myBlock)(void) = ^{
			NSLog(@"myblock");
    };
    myBlock();
    return 0;
}
```

转成C代码后，产生了十万多行代码，只有下面几行是相关逻辑，下面介绍下主要实现逻辑

### Block的数据结构

我们这个block没有任何参数，没有返回值，也没有变量捕获，下面看看它的数据结构

````c
struct __main_block_impl_0 {
  //函数实现的结构
  struct __block_impl impl;
  //描述
  struct __main_block_desc_0* Desc;
  //构造函数
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    //stack block
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
````

上面提到的`__block_impl`的结构如下

```c
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
```

同样是一个结构体，有isa指针，flag，保留字段，以及函数指针。

另外还有一个block描述结构，这个结构题只有block的size，以及保留字段。

````c
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
````

这就是一个极简block的全部结构了。

* __block_impl中存储函数指针
* _main_block_desc_0 中存储block的size信息

### 构造与执行block

有了数据结构，下面就可以看下如何使用的。main方法中有两句话，一个是构造，一个是执行。

```c
int main(int argc, const char * argv[]) {
    //构造
    void (*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    //执行
    ((void (*)(__block_impl *))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock);
    return 0;
}
```

可以看下上面提到的这个结构题的构造函数

````c
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) 
````

需要一个函数指针，一个block的描述，一个flag，main函数中传入了`__main_block_func_0`函数，这个函数定义如下

````c
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_81_ndch2_f14qn4l3v4ygy_mdtw0000gn_T_main_8dbd0a_mi_0);
}
````

这个函数很简单，就是一个nslog打印语句。

在往下就是执行该block，实际是执行了函数指针指向的函数。

## 添加参数与返回值

### 数据结构

数据结构没有变化

### 构造与执行block

````c
static int __main_block_func_0(struct __main_block_impl_0 *__cself, int a) {
    return a + 2;
}

int main(int argc, const char * argv[]) {
  	//构造
    int (*myBlock)(int a) = ((int (*)(int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
  	//执行
    int b = ((int (*)(__block_impl *, int))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock, 1);
    printf("%d", b);
    return 0;
}
````

这里变化还是很多的，首先，用来填充函数指针的，`__main_block_func_0`参数增加了一个int类型的a，以及int类型的返回值，与我们的block一致。构造和执行也均有相关的调整。

## 继续添加一个自由变量的捕获

### 数据结构

`__main_block_desc_0`与`__block_impl`结构没有发生变化，`__main_block_impl_0`增加了一个字段

````c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int c;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _c, int flags=0) : c(_c) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
````

被block捕获的变量，存在了这个表示block的结构体中。值传递的方式，通过构造函数传入。

### 构造与执行

````c
static int __main_block_func_0(struct __main_block_impl_0 *__cself, int a) {
    int c = __cself->c; // bound by copy
    return a + c;
}

int main(int argc, const char * argv[]) {
    int c = 0;
    int (*myBlock)(int a) = ((int (*)(int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, c));
    int b = ((int (*)(__block_impl *, int))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock, 1);
    printf("%d", b);
    return 0;
}
````

`__main_block_func_0`函数表示block逻辑，这里可以看到，从结构体中取出c，与传入的参数a想加返回。

`main`函数中，用c的值构造该block的数据结构。从c的传入方式来看，内部c的值跟外部是没有关系的。



## __block的自由变量

### 数据结构

数据结构变化还是挺大的，首先被`__block`修饰的变量c，被包装成了一个结构体。

```c
struct __Block_byref_c_0 {
  void *__isa;
__Block_byref_c_0 *__forwarding;
 int __flags;
 int __size;
 int c;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_c_0 *c; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_c_0 *_c, int flags=0) : c(_c->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->c, (void*)src->c, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->c, 8/*BLOCK_FIELD_IS_BYREF*/);}
```

block对应的结构体中的c现在是这个类型的实例。`__Block_byref_c_0`中有个`forwarding`，指向一个自身的实例。这个很重要，是实现内部修改，外部可感知，或者外部修改，可以在内部感知的关键。

另外`__main_block_desc_0`中增加了copy和dispose。因为有`__block`的变量，会导致block从栈空间，拷贝到堆空间。对应的就要提供copy以及dispose（清理）的逻辑。

当block拷贝到堆区，栈区中的c和堆区中的c的forwarding就会指向堆区中的c。后续的操作，都是如下方式访问

```c
c->forwarding->c
```

这样，不管是栈区的c还是堆区的c修改，都会修改堆区的c，访问也是一样。

### 构造与执行

````c
static int __main_block_func_0(struct __main_block_impl_0 *__cself, int a) {
  //取出c
    __Block_byref_c_0 *c = __cself->c; // bound by ref
  //拿出c指向的__Block_byref_c_0的c，做++操作
    (c->__forwarding->c)++;
    return a + (c->__forwarding->c);
}

int main(int argc, const char * argv[]) {
	//构造__Block_byref_c_0,forwarding是该实例的指针。
    __attribute__((__blocks__(byref))) __Block_byref_c_0 c = {(void*)0,(__Block_byref_c_0 *)&c, 0, sizeof(__Block_byref_c_0), 4};
		//构造block对象的结构	
    int (*myBlock)(int a) = ((int (*)(int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_c_0 *)&c, 570425344));
		//执行该结构保存的函数指针
    int b = ((int (*)(__block_impl *, int))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock, 1);
    printf("%d", b);
    return 0;
}
````

了解了`__Block_byref_c_0`的使用，那么这部分的执行跟之前也没什么太大区别。

