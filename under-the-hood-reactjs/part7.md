# Part 7

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/7/part-7.svg)

## 内容

### `markup` 对象

在 `ReactMount` 中，生成了挂载对象 `markup`

```js
// src/renderers/dom/client/ReactMount.js#109
var markup = ReactReconciler.mountComponent(
  wrapperInstance,
  transaction,
  null,
  ReactDOMContainerInfo(wrapperInstance, container),
  context,
  0 /* parentDebugID */,
);
```

这个对象不是 HTML node，而是一个包含了 `children` 和 `node` 属性的数据结构，其中 `node` 属性指出了待插入 HTML 中的 DOM 节点。

之后，通过 ，进行节点插入：

```js
// src/renderers/dom/client/ReactMount.js#123
ReactMount._mountImageIntoNode(
  markup,
  container,
  wrapperInstance,
  shouldReuseMarkup,
  transaction,
);
```

在 `ReactMount._mountImageIntoNode()` 方法中，我们看到核心代码：

```js
// src/renderers/dom/client/ReactMount.js#742
if (transaction.useCreateElement) {
  while (container.lastChild) {
    container.removeChild(container.lastChild);
  }
  DOMLazyTree.insertTreeBefore(container, markup, null);
} else {
  setInnerHTML(container, markup);
  ReactDOMComponentTree.precacheNode(instance, container.firstChild);
}
```

在组件对应的 DOM 节点插入到我们声明的 `container` 之前，会：

1. 清空 container 内的节点
2. 通过 `DOMLazyTree.insertTreeBefore()` 方法（该方法进行节点插入时使用的 API 是 [`Node.insertBefore`]https://developer.mozilla.org/en-US/docs/Web/API/Node/insertBefore()），将节点插入

### 挂载之后

前面部分，已经提过，在组件挂载之后，借由事务，还会进行如下操作：

- 通过 `ReactInputSelection.restoreSelection()` 恢复输入选取。
- 通过 `ReactBrowserEventEmitter.setEnabled(previouslyEnabled)` 恢复事件监听。
- 通过 `this.reactMountReady.notifyAll` 来唤起所有 `transaction.reactMountReady` 队列中的回调函数，回调中最重要，也最为我们熟知的是 `componentDidMount()` 函数。

由于挂载过程使用了 `ReactDefaultBatchingStrategyTransaction` 事务，其 Wrapper 列表中的 `FLUSH_BATCHED_UPDATES` 的关闭方法为 `ReactUpdates.flushBatchedUpdates()`，因此，挂载之后，还将调用该方法来处理 `dirtyComponents`（只是目前还没有 `dirtyComponents` 需要我们处理）。

## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/7/part-7-B.svg)

本质上：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/7/part-7-C.svg)

目前，React 中挂载的流程我们也算走完了：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/7/mounting-parts-C.svg)