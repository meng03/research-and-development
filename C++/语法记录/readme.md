# 随手记录一些C++语法

## 创建对象与构造函数

创建对象有很多方式

````
Person person = Person(name: "张三");
Person person = new Person(name: "张三");
Person person(name: "张三");
````

构造函数大部分都与其他语言类似，但是有一种用，初始化列表来吃时候字段的，比较特殊

````c++
//oc runtime库,isa_t结构体的部分定义
union isa_t 
{
    isa_t() { }
    //使用初始化列表来初始化字段
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
}
````

另一个简单的例子

```c++
Line::Line( double len): length(len) {}
//等同于
Line::Line( double len)
{
    length = len;
}

```

