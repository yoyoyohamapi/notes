# Part 10

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/10/part-10.svg)

## 内容

### Dirty Components（脏组件）与 `ReactUpdatesFlushTransaction`

当我们的组件发生了状态变动后，该组件就会被标识为脏组件，`ReactUpdatesFlushTransaction` 事务将完成清除脏组件的任务，并在之后执行已入队的更新操作（如 `componentDidUpdate`）：

```js
if (dirtyComponents.length) {
  var transaction = ReactUpdatesFlushTransaction.getPooled();
  transaction.perform(runBatchedUpdates, null, transaction);
  ReactUpdatesFlushTransaction.release(transaction);
}
```

该事务的 Wrapper 列表为 `[NESTED_UPDATES, UPDATE_QUEUEING]`：

```js
var NESTED_UPDATES = {
  initialize: function() {
    this.dirtyComponentsLength = dirtyComponents.length;
  },
  close: function() {
    if (this.dirtyComponentsLength !== dirtyComponents.length) {
      // Additional updates were enqueued by componentDidUpdate handlers or
      // similar; before our own UPDATE_QUEUEING wrapper closes, we want to run
      // these new updates so that if A's componentDidUpdate calls setState on
      // B, B will update before the callback A's updater provided when calling
      // setState.
      dirtyComponents.splice(0, this.dirtyComponentsLength);
      flushBatchedUpdates();
    } else {
      dirtyComponents.length = 0;
    }
  },
};

var UPDATE_QUEUEING = {
  initialize: function() {
    this.callbackQueue.reset();
  },
  close: function() {
    this.callbackQueue.notifyAll();
  },
};
```

因此，我们知道，事务初始化时，将完成：

1. 缓存脏组件列表长度
2. 重置更新回调队列

事务关闭时，将完成：

1. 观察脏组件列表是否更新，如果没有更新，则结束。否则继续通过 `flushBatchedUpdates()` 更新脏组件列表。
2. 通知所有更新回调队列上的方法

另外，在 `ReactUpdatesFlushTransaction.perform()` 中，嵌套使用了 `ReactReconcileTransaction`，因此，这个事务的结构更具体地是：

```
[NESTED_UPDATES, UPDATE_QUEUEING].initialize()
[SELECTION_RESTORATION, EVENT_SUPPRESSION, ON_DOM_READY_QUEUEING].initialize()

method -> ReactUpdates.runBatchedUpdates

[SELECTION_RESTORATION, EVENT_SUPPRESSION, ON_DOM_READY_QUEUEING].close()
[NESTED_UPDATES, UPDATE_QUEUEING].close()
```

再看到该事务服务的方法 —— `runBatchedUpdates`，顾名思义，它将完成脏组件的更新：

1. 对脏组件按照挂载顺序进行排序，这意味着先挂载的先更新：

```js
// src/renderers/shared/stack/reconciler/ReactUpdates.js#132
dirtyComponents.sort(mountOrderComparator);
```

2. 刷新更新计数，从注释中我们知道这么做的原因是避免组件的重复更新：

```js
// src/renderers/shared/stack/reconciler/ReactUpdates.js#139 
updateBatchNumber++;
```

3. 迭代各个脏组件，如果有必要，则更新：

```js
// src/renderers/shared/stack/reconciler/ReactUpdates.js#141
for (var i = 0; i < len; i++) {
  var component = dirtyComponents[i];

  var callbacks = component._pendingCallbacks;
  component._pendingCallbacks = null;

  // ...

  ReactReconciler.performUpdateIfNecessary(
    component,
    transaction.reconcileTransaction,
    updateBatchNumber,
  );

  // ...
  if (callbacks) {
    for (var j = 0; j < callbacks.length; j++) {
      transaction.callbackQueue.enqueue(
        callbacks[j],
        component.getPublicInstance(),
      );
    }
  }
}
```

## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/10/part-10-B.svg)

本质上：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/10/part-10-C.svg)