# Aspects 源码阅读

问题

> A method can only be hooked once per class hierarchy.

为什么一个类层级中，一个方法只能被hook一次。

替换hook被多次hook会有影响。

前后添加逻辑的hook影响小一点



`object_getClass`，`self.class`与`[self class]` 区别?



