# Part 9

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/9/part-9.svg)

## 内容

考虑到一个用例，我们点击按钮来设置组件状态：

```jsx
<button
  onClick = {() => this.setState({message: 'from click'})}
>
  Click Me!
</button>
```

React 通过 `ReactEventListener.dispatchEvent()` 进行事件处理：

```js
// src/renderers/dom/client/ReactEventListener.js#153
dispatchEvent: function(topLevelType, nativeEvent) {
  if (!ReactEventListener._enabled) {
    return;
  }

  var bookKeeping = TopLevelCallbackBookKeeping.getPooled(
    topLevelType,
    nativeEvent,
  );
  try {
    // Event queue being processed in the same cycle allows
    // `preventDefault`.
    ReactUpdates.batchedUpdates(handleTopLevelImpl, bookKeeping);
  } finally {
    TopLevelCallbackBookKeeping.release(bookKeeping);
  }
}
```

1. 如果 `ReactEventListener` 没有被激活，则取消事件分发。前面内容我们讨论过，如果组件在挂载中，会取消事件监听（通过 `ReactReconcileTransaction` 中的某个 Wrapper）。
2. 使用 `ReactUpdates.batchedUpdates` 来批量处理事件，并且确定事务的开启，在其中，我们的组件实例会被放入 `dirtyComponents` 列表
3. 事务关闭时，会调用 `ReactUpdates.flushBatchedUpdates()` 来处理 `dirtyComponents`

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/9/set-state-update-start.svg)


## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/9/part-9-B.svg)

本质上：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/9/part-9-C.svg)

# 补充

React 事件系统概览：

```js
/*
 * Overview of React and the event system:
 *
 * +------------+    .
 * |    DOM     |    .
 * +------------+    .
 *       |           .
 *       v           .
 * +------------+    .
 * | ReactEvent |    .
 * |  Listener  |    .
 * +------------+    .                         +-----------+
 *       |           .               +--------+|SimpleEvent|
 *       |           .               |         |Plugin     |
 * +-----|------+    .               v         +-----------+
 * |     |      |    .    +--------------+                    +------------+
 * |     +-----------.--->|EventPluginHub|                    |    Event   |
 * |            |    .    |              |     +-----------+  | Propagators|
 * | ReactEvent |    .    |              |     |TapEvent   |  |------------|
 * |  Emitter   |    .    |              |<---+|Plugin     |  |other plugin|
 * |            |    .    |              |     +-----------+  |  utilities |
 * |     +-----------.--->|              |                    +------------+
 * |     |      |    .    +--------------+
 * +-----|------+    .                ^        +-----------+
 *       |           .                |        |Enter/Leave|
 *       +           .                +-------+|Plugin     |
 * +-------------+   .                         +-----------+
 * | application |   .
 * |-------------|   .
 * |             |   .
 * |             |   .
 * +-------------+   .
 * /
```