# Part 8

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/8/part-8.svg)

## 内容

### `ReactComponent.prototype.setState`

在 React 中，我们最常使用 `setState(partialState, callback)` 来进行状态更新，从而更新我们的组件：

```js
this.setState({message: 'hello'})
```

由于我们的组件都是继承自 `React.Component`，所以我们呢可以看到其原型上的 `setState()` 方法：

```js
// src/isomorphic/modern/class/ReactBaseClasses.js#58
ReactComponent.prototype.setState = function(partialState, callback) {
  // ...
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState');
  }
};
```

可以看到，我们是使用对应组件实例的 `updater` 更新器将状态更新操作送入更新队列的，深入到源码：

```js
// src/renderers/shared/stack/reconciler/ReactUpdateQueue.js#230
enqueueSetState: function(publicInstance, partialState) {

  var internalInstance = getInternalInstanceReadyForUpdate(
    publicInstance,
    'setState',
  );

  if (!internalInstance) {
    return;
  }

  var queue =
    internalInstance._pendingStateQueue ||
    (internalInstance._pendingStateQueue = []);
  queue.push(partialState);

  enqueueUpdate(internalInstance);
}
```

> 其中，`internalInstance` 指代的是 `ReactCompositeComponent` 实例，而 `publicInstance` 指代的是我们自定义的 `ExampleApplication`。

将状态更新操作入队将经历如下几个步骤：

1. 根据 `publicInstance`，从 `ReactInstanceMap` 拿到中 React Component 实例

```js
// src/renderers/shared/stack/reconciler/ReactUpdateQueue.js#38
var internalInstance = ReactInstanceMap.get(publicInstance);
```

2. 将待更新的状态子树放入组件实例的待更新状态队列中

```js
// src/renderers/shared/stack/reconciler/ReactUpdateQueue.js#240
var queue =
  internalInstance._pendingStateQueue ||
  (internalInstance._pendingStateQueue = []);
queue.push(partialState);
```

3. 通过 `ReactUpdates.enqueueUpdate(internalInstance)` 来检查是否更新过程已经就绪：

```js
// src/renderers/shared/stack/reconciler/ReactUpdates.js
function enqueueUpdate(component) {
  ensureInjected();

  if (!batchingStrategy.isBatchingUpdates) {
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }

  dirtyComponents.push(component);
  if (component._updateBatchNumber == null) {
    component._updateBatchNumber = updateBatchNumber + 1;
  }
}
```

如果没有在批量更新，先通过 `batchingStrategy.batchedUpdates()` 开启批量更新事务，否则将组件实例放入 `dirtyComponents` 序列中。

## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/8/part-8-B.svg)

本质上：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/8/part-8-C.svg)

## 补充

`ReactDefaultBatchingStrategyTransaction`
