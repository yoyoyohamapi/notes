# Part-3

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3.svg)

## 内容

### `mountComponent()`

当 `TopLevelWrapper` 挂载完毕后，就开始挂载它包裹的子组件，也就是我们的 `<ExampleApplication />`，接着上一部分的 `ReactReconciler.mountComponent` 我们知道，组件的挂载行为还是交给了组件对象自身：

```js
// src/renderers/shared/stack/reconciler/ReactReconciler.js#25
var ReactReconciler = {
  // ...
  mountComponent: function() {
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

由于 `<ExampleApplication />` 是 React Composite Component，因此，我们看到 `src/renderers/shared/stack/reconciler/ReactCompositeComponent.js`：

```js
var ReactCompositeComponent = {
  // ...

  /**
   * 初始化组件，渲染组件构成，为组件注册时间监听
   */
  mountComponent: function() {

    // ...
    var updateQueue = transaction.getUpdateQueue();

    // ...
    var inst = this._constructComponent();
   
    // ...
    
    inst.props = publicProps;
    inst.context = publicContext;
    inst.refs = emptyObject;
    inst.updater = updateQueue;

    this._instance = inst;

    ReactInstanceMap.set(inst, this);

    // ...

    var initialState = inst.state;
    if (initialState === undefined) {
      inst.state = initialState = null;
    }

    this._pendingStateQueue = null;
    this._pendingReplaceState = false;
    this._pendingForceUpdate = false;

    // ...
    markup = this.performInitialMount();

    // ...
    transaction.getReactMountReady().enqueue(inst.componentDidMount, inst);

    return markup;
  }
}
```

通过 `_constructComponent()` 方法，构造了 `<ExmapleApplication />` 组件的实例化对象：

```js
ExampleApplication: {
  props: {
    // ...
  },
  context: undefined,//
  refs: {
    // ...
  },
  updater: {
    // ...
  },
  state: {
    // ...
  }
}
```

由于 `ReactCompositeComponent` 是抽象给所有平台的，因此需要在挂载过程中，为其动态绑定其需要的更新器 `updater`，这里 `updater` 来自于 `transaction.getUpdateQueue()`，是 `ReactUpdateQueue` 模块。 

### `performInitialMount()`

接下来，通过 `performInitialMount()` 开始执行初始化挂载：

```js
var ReactCompositeComponent = {
  // ...
  performInitialMount: function ) {

    inst.componentWillMount();
    // When mounting, calls to `setState` by `componentWillMount` will set
    // `this._pendingStateQueue` without triggering a re-render.
    if (this._pendingStateQueue) {
      inst.state = this._processPendingState(inst.props, inst.context);
    }

    // If not a stateless component, we now render
    if (renderedElement === undefined) {
      renderedElement = this._renderValidatedComponent();
    }

    var nodeType = ReactNodeTypes.getType(renderedElement);
    this._renderedNodeType = nodeType;
    var child = this._instantiateReactComponent(
      renderedElement,
      nodeType !== ReactNodeTypes.EMPTY /* shouldHaveDebugID */,
    );
    this._renderedComponent = child;

    var markup = ReactReconciler.mountComponent );
    return markup;
  },
}
```

在这个流程中，将会执行我们申明的生命期钩子 `componentWillMount()`，并且从代码中可以看到，如果我们在 `componentWillMount()` 中进行了 `setState()` ，即此时状态变更队列 `this._pendingStateQueue` 非空，那么不会触发组件的重渲染，而是直接更新组件实例的状态。在初始化挂载阶段，我们又遇到 `ReactReconciler.mountComponent`，此时，它将负责挂载子组件实例。

`performIntialMount()` 执行完成，回到 `mountComponent` 方法，事务将会让组件的 `componentDidMount()` 钩子进入回调队列：

```js
transaction.getReactMountReady().enqueue(inst.componentDidMount, inst);
```

## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3-B.svg)

实质：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/3/part-3-C.svg)