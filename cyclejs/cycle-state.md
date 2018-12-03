# Cycle.js 状态管理模型

## 分型（fractal）

现在前端领域，最流行的状态管理模型毫无疑问是 redux，但遗憾的是，redux 并不是一个分型架构。什么是分型架构：

> 如果子组件能够以同样的结构，作为一个应用使用，这样的结构就是分型架构。

在分型架构下，每个组件都可以作为应用独立使用，而在非分型架构下，每个组件只是整个应用的一部分，往往依赖于一个统揽全局的**协调器（orchestrators）**。例如我们说到的 redux，redux 只是聚焦于状态管理，而不涉及组件的视图实现，视图（View）是交由组件自己实现的，而不是一个完整的应用闭环，因此 redux 不是一个分型架构，在 redux 中，协调器指的就是 `Store` 对象。

![Redux diagram](https://staltz.com/img/redux-unidir-ui-arch.jpg)

我们再看下作为 redux 灵感来源的 Elm：

![Model-View-Update diagram](https://staltz.com/img/mvu-unidir-ui-arch.jpg)

在 Elm 架构，下，我们看到，每个组件都有一个完整的应用闭环：

- 一个 Model 类型
- 一个 Model 的初始实例
- 一个 View 函数
- 一个 Action type 以及对应的更新函数

因此，Elm 就是分型架构的，每个 Elm 组件也是一个 Elm 应用。

## 参考资料

- [UNIDIRECTIONAL USER INTERFACE ARCHITECTURES](https://staltz.com/unidirectional-user-interface-architectures.html)