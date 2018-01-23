# Part-1
 
![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1.svg)

## 内容

### Transaction（事务）

`ReactUpdates` 模块能够将上一章中初始化的 React component 实例**连接**到 React 的生态中，React 是**成块（chunk）更新的**，并且会为一系列的 chunk 应用一些**前置条件（pre-condition）**和**后置条件（post-condition）**，因此引入了事务这个模式。

以下面的通信信道为例，非事务模式下，**每个**消息的传输都会开关一次连接，而在事务模式下，**批量的消息**才开关一次连接：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/communication-channel.svg)

React 中事务的作用主要有：

- 使用前置及后置条件包裹行为
- 允许应用程序重置事务流
- 如果一个事务已经在运行了，锁住其他同时执行的事务

在 React 中，存在多个事务类服务于不同行为，它们都继承于 `Transaction` 模块。各个事务类都有其 Wrapper 列表，因此形成它们间的差异性，各个 Wrapper 是包含了事务初始化和事务关闭方法的对象。例如在 `ReactDefaultBatchingStrategyTransaction` 中，其 Wrapper 序列就有 `RESET_BATCHED_UPDATES` 就 `FLUSH_BATCHED_UPDATES`：

```js
// \src\renderers\shared\stack\reconciler\ReactDefaultBatchingStrategy.js#19
var RESET_BATCHED_UPDATES = {
	  initialize: emptyFunction,
	  close: function() {
		ReactDefaultBatchingStrategy.isBatchingUpdates = false;
	  },
};

var FLUSH_BATCHED_UPDATES = {
	 initialize: emptyFunction,
	 close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
}

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];
```

所以，使用事务执行某个方法就可以概括为：

- 调用该事务 Wrapper 列表中的每个初始化方法，这些方法的执行结果将被缓存以供后续使用。
- 执行事务包裹的方法
- 调用该事务 Wrapper 列表中的每个关闭方法来关闭事务

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/transaction.svg)

在上述的 `ReactDefaultBatchingStrategyTransaction` 事务中，没有前置条件，而在 `FLUSH_BATCHED_UPDATES` 这个 Wrapper 中，其关闭事务时，会对将要再次渲染的脏组件开启验证。将挂载任务包裹在这个 Wrapper 中是因为挂载之后，React 需要检查挂载后的组件造成的影响，并且批量更新它们。

React 中的事务服务于以下用例：

- 保存 reconciliation 前后的 selection range。
- 当 DOM 重排时，冻结事件，防止 focus/blur，并在重排完成后，重新激活这些事件。
- 当 reconciliation 发生在一个 worker 线程时，清空 DOM 对主 UI 线程产生的变更序列。
- 在新的内容渲染后，唤起各个 `componentDidUpdate` 回调。

## 总结

Part-1 可以概括为：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1-B.svg)

实质：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1-C.svg)

## 补充

1. 校验与阻断

在 React 源码中，大量使用了 [invariant]() 这个 package，它提供了一个方法做真值检测，当测试通过，则执行后续流程，否则输出错误信息：

```js
var invariant = require('invariant');
 
invariant(someTruthyVal, 'This will not throw');
// No errors 
 
invariant(someFalseyVal, 'This will throw an error with this message');
// Error: Invariant Violation: This will throw an error with this message
```