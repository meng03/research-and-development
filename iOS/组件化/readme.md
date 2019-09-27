# 组件化

组件化目的

* 梳理项目，将繁杂的模块间调用梳理清楚

* 方便组件独立维护，编译更快，没有合并代码问题，更适合大团队

* 方便组件复用，例如一个业务线的多个App的登录模块，以及用户模块可能是一样的





要解决的问题

* 模块需要App的全部状态变化

* 如何解耦

* 如何实现组件间通信



现有方案

* 中间人调度

* url-block

* Protocol-class



面向协议，依赖注入

模块内调用，模块外调用，外部调用（push，universal link，js，server response，由其他App打开）



是否可以考虑由底层模块实现一个暂存逻辑，当需要传输大型对象，图片，Data等，可以先将这个对象存入这个暂存模块，然后将key传给被调方，被调方发现有这个key时，就去暂存模块中取出数据

## 中间人调度方案

问题

1. Mediator 怎么去转发组件间调用？
2. 一个模块只跟 Mediator 通信，怎么知道另一个模块提供了什么接口？
3. 按上图的画法，模块和 Mediator 间互相依赖，怎样破除这个依赖？

粗略的想法：

设计一个数据结构（request），由组件构建，交给Mediator去处理。这个request的核心信息，由后台下发，比如在一个cell的模型信息中添加一个request结构，用来做点击的响应逻辑。

request先交给自己模块的路由来处理，自己模块的路由发现无法处理，就交给Mediator来处理（跨组件处理）。

mediator根据注册的组件信息，将request分发出去，比如request中有个字段叫Module，Module拿到request，就处理request。

问题：

如何实现转场动画

如果后台不愿意下发request信息，是否要通过注册表来实现

思考

如果通过通知发出去，是不是连Mediator都省了，ModuleName作为NotificationName，每个模块的Router监听各自的通知，

通知的坏处是没有一个统一的处理，出现问题，无法统一拦截。

那还是Mediator更合适



casa 给出了此方案的详细逻辑以及实现代码，主要是基于Mediator模式和Target-Action模式，中间采用了runtime来完成调用。

###



组件化容器（ModuleAppDelegate），同步主工程App状态

问题：

组件拆分之后，后续的一些小功能可能会各个组件自己实现，时间长了，多个组件就会有很多功能相似的代码，如何处理？

将这些可能会成为工具的方法写在一个预工具包中，每个月整理一次，放到底部库中





参考：

<https://blog.cnbluebox.com/blog/2015/11/28/module-and-decoupling/>

<https://zhuanlan.zhihu.com/p/22565656>

<https://limboy.me/tech/2016/03/14/mgj-components-continued.html>