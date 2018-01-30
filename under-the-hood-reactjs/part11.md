# Part 11

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/11/part-11.svg)

## 内容

在 `ReactUpdates.runBatchedUpdates()` 中，会迭代我们的脏组件，并且调用 `ReactReconciler.performUpdateIfNecessary()` 进行组件更新。类似 `ReactReconciler.mountComponent()`，该方法最终还是调用的组件实例上的 `performUpdateIfNecessary()` 进行组件更新的。

因此，我们可以看到 `ReactCompositeComponent.performUpdateIfNecessary()` 方法， 当存在状态更新队列不为空或者用户调用过 `this.forceUpdate()` 方法，就进行组件更新：

### 组件的更新必要性

```js
// src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#750
performUpdateIfNecessary: function(transaction) {
  if (this._pendingElement != null) {
    ReactReconciler.receiveComponent(
      this,
      this._pendingElement,
      transaction,
      this._context,
    );
  } else if (this._pendingStateQueue !== null || this._pendingForceUpdate) {
    this.updateComponent(
      transaction,
      this._currentElement,
      this._currentElement,
      this._context,
      this._context,
    );
  } else {
    this._updateBatchNumber = null;
  }
}
```

### 判断更新

通过 `ReactCompositeComponent.updateComponent()` 方法，我们看下 React 是怎么进行组件更新的：

1. 设置更新前后属性：

```js
var prevProps = prevParentElement.props;
var nextProps = nextParentElement.props;
```

2. 如果是属性更新，调用生命期方法 `componentWillReceiveProps`：

```js
inst.componentWillReceiveProps(nextProps, nextContext);
```

3. 从状态设置队列上取出下一次状态：

```js
var nextState = this._processPendingState(nextProps, nextContext);
```

4. 如果设置了 `shouldComponentUpdate`，则通过这个钩子判断是否需要更新，否则如果是 PureComponent，则对更新前后的新旧状态、属性进行浅比较判断，React 默认认为需要更新：

```js
var shouldUpdate = true;

if (inst.shouldComponentUpdate) {
  shouldUpdate = inst.shouldComponentUpdate(
    nextProps,
    nextState,
    nextContext,
  );
} else {
  if (this._compositeType === CompositeTypes.PureClass) {
    shouldUpdate =
      !shallowEqual(prevProps, nextProps) ||
      !shallowEqual(inst.state, nextState);
  }
}
```

5. 如果需要更行，则通过 `ReactCompositeComponent._performComponentUpdate()` 进行更新。另外，即便不需要要更新，还是会绑定后下一次的 `props`、`state` 和 `context` 到当前的组件实例上：

```js
if (shouldUpdate) {
  this._pendingForceUpdate = false;
  this._performComponentUpdate(
    nextParentElement,
    nextProps,
    nextState,
    nextContext,
    transaction,
    nextUnmaskedContext,
  );
} else {
  this._currentElement = nextParentElement;
  this._context = nextUnmaskedContext;
  inst.props = nextProps;
  inst.state = nextState;
  inst.context = nextContext;
}
```
### 执行更新

接下来，从 `ReactCompositeComponent._performComponentUpdate()` 方法来了解 React 组件的更新流程：

1. 如果声明了 `componentDidMount` 钩子，则缓存当前属性（亦即尚未更新的属性）为前一次属性：

```js
var hasComponentDidUpdate = Boolean(inst.componentDidUpdate);
var prevProps;
var prevState;
var prevContext;
if (hasComponentDidUpdate) {
  prevProps = inst.props;
  prevState = inst.state;
  prevContext = inst.context;
}
```

2. 如果声明了 `componentWillMount` 钩子，则执行：

```js
if (inst.componentWillUpdate) {
  inst.componentWillUpdate(nextProps, nextState, nextContext);
}
```

3 . 刷新 React Element 及组件属性：

```js
this._currentElement = nextElement;
this._context = unmaskedContext;
inst.props = nextProps;
inst.state = nextState;
inst.context = nextContext;
```

### 组件再渲染

之后，便通过 `ReactCompositeComponent._updatedRenderedCompoent()` 方法渲染组件，更新 DOM：

1. 缓存当前渲染的 React Element：

```js
var prevComponentInstance = this._renderedComponent;
var prevRenderedElement = prevComponentInstance._currentElement;
```

2. 通过 `ReactCompositeComponent._renderValidatedComponent()` 调用组件实例的 `render()` 方法，获得渲染后的组件：

```js
var nextRenderedElement = this._renderValidatedComponent();
```

3. 借由 `shouldUpdateReactComponent` 模块，判断当前组件实例是应当更新？销毁？还是重新创建一个新的实例：

```js
if (shouldUpdateReactComponent(prevRenderedElement, nextRenderedElement)) {
  // ...
} 
```

4. 如果需要更新，则通过 `ReactReconciler.receiveComponent` 更新组件，刷新 DOM。其内部同样是调用的组件实例（`prevComponentInstance`）的 `receiveComponent()` 方法：

```js
ReactReconciler.receiveComponent(
  prevComponentInstance,
  nextRenderedElement,
  transaction,
  this._processChildContext(context),
);
```

##总结 

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/11/part-11-B.svg)

本质上：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/11/part-11-C.svg)

## 补充

### 装饰器

在 React 源码中，我们看到，生命周期钩子的调用都使用了一个函数装饰器 `measureLifeCyclePerf`，借助于装饰器，我们可以在函数运行前后完成一些额外工作，比如这个例子中，就是在调试时通过 debug 工具进行性能标记： 

```js
function measureLifeCyclePerf(fn, debugID, timerType) {
  if (debugID === 0) {
    // Top-level wrappers (see ReactMount) and empty components (see
    // ReactDOMEmptyComponent) are invisible to hooks and devtools.
    // Both are implementation details that should go away in the future.
    return fn();
  }

  ReactInstrumentation.debugTool.onBeginLifeCycleTimer(debugID, timerType);
  try {
    return fn();
  } finally {
    ReactInstrumentation.debugTool.onEndLifeCycleTimer(debugID, timerType);
  }
}
```

