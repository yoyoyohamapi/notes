# Part 2

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/2/part-2.svg)

## 内容

### 又一个事务

在 `ReactMount._renderNewRootComponent` 中，有：

```js
// src/renderers/dom/client/ReactMount.js#391
ReactUpdates.batchedUpdates(
  batchedMountComponentIntoNode,
  componentInstance,
  container,
  shouldReuseMarkup,
  context,
);
```

`ReactUpdates.batchedUpdates()` 接收的第一个参数是一个回调函数，供注入的批量更新策略 `batchingStrategy` 使用：

```js
// src/renderers/shared/stack/reconciler/ReactUpdates.js#103
function batchedUpdates(callback, a, b, c, d, e) {
  ensureInjected();
  return batchingStrategy.batchedUpdates(callback, a, b, c, d, e);
}
```

这里使用的回调是 `batchedMountComponentIntoNode()`，亦即更新之后，将组件挂载到对应节点中：

```js
// src/renderers/dom/client/ReactMount.js#391
function batchedMountComponentIntoNode(
  componentInstance,
  container,
  shouldReuseMarkup,
  context,
) {
  var transaction = ReactUpdates.ReactReconcileTransaction.getPooled(
    /* useCreateElement */
    !shouldReuseMarkup && ReactDOMFeatureFlags.useCreateElement,
  );
  transaction.perform(
    mountComponentIntoNode,
    null,
    componentInstance,
    container,
    transaction,
    shouldReuseMarkup,
    context,
  );
  ReactUpdates.ReactReconcileTransaction.release(transaction);
}
```

在这里，我们看到了另一个事务 `ReactReconcileTransaction`，它将服务于组件挂载，按照 React 的事务约定，它也有一系列的 Wrapper：

```js
//\src\renderers\dom\client\ReactReconcileTransaction.js#89
var TRANSACTION_WRAPPERS = [
  SELECTION_RESTORATION,
  EVENT_SUPPRESSION,
  ON_DOM_READY_QUEUEING,
];
```

因此，这个事务在初始化（`initialize()`）时会：

1. 保存输入选取
2. 禁用事件
3. 初始化 `onDOMReady` 队列

执行完成 `mountComponentIntoNode` 后，将会（`close()`）：

1. 恢复输入选取
2. 恢复事件监听
3. 唤起 `onDOMReady` 队列上的各个回调

再看到该事务包裹的方法 `mountComponentIntoNode()`：

```js
// src/renderers/dom/client/ReactMount.js#92
function mountComponentIntoNode() {
  // ...
  var markup = ReactReconciler.mountComponent(
    wrapperInstance,
    transaction,
    null,
    ReactDOMContainerInfo(wrapperInstance, container),
    context,
    0 /* parentDebugID */,
  ); 
  // ...
}
```

但是，该方法只能算是一个 Wrapper，因为组件的挂载是平台相关的，而 `ReactReconciler` 经常服务于依赖于平台的逻辑：

```js
// src/renderers/shared/stack/reconciler/ReactReconciler.js#25
var ReactReconciler = {
  // ...
  mountComponent: function(
    internalInstance,
    transaction,
    hostParent,
    hostContainerInfo,
    context,
    parentDebugID, // 0 in production and for roots
  ) {
    // ...
    var markup = internalInstance.mountComponent(
      transaction,
      hostParent,
      hostContainerInfo,
      context,
      parentDebugID,
    );
    if (
      internalInstance._currentElement &&
      internalInstance._currentElement.ref != null
    ) {
      transaction.getReactMountReady().enqueue(attachRefs, internalInstance);
    }
    // ...
    return markup;
  }
  // ...
}
```

可以看到，组件的挂载还是委托给了组件自身实例，挂载过程也是我们知道的：初始化组件 --> 渲染组件组成 --> 为组件注册事件。在我们的 deme 中，上述代码中依次进行挂载的 `internalInstance` （React Component 实例）是：

```js
ReactCompositeComponentWrapper(TopLevelWrapper)
ReactCompositeComponentWrapper(<ExampleApplication />)
ReactDOMComponent(<div />)
ReactDOMComponent(<button />)
ReactCompositeComponentWrapper(<ChildCmp />)
ReactDOMComponent(<div />)
ReactDOMTextComponent("And some text as well!")
```

## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/2/part-2-B.svg)

实质：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/2/part-2-C.svg)