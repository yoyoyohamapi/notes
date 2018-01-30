# Part 0

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/0/part-0.svg)

## 内容

### JSX

React 做的是 View（视图）层的事儿，它让我们使用 JSX 去定义一个组件，指定其属性和行为，先抛开诸如 Redux 这样的状态管理模式不谈，多个 React 组合在一起，就构成了我们的应用。因此，React 的工作流程可以简单的用下面的线性流程概括：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/0/mounting-scheme-1-small.svg)

更具体地：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/0/mounting-scheme-1-big.svg)

### React Element

首先着眼于 Demo 中的第一个函数调用 `ReactDOM.render`，这算是我们的组件入口，该方法最终将我们的组件渲染到页面的 DOM 树中。我们使用 JSX 来对组件进行**高级抽象**，而浏览器不会认可这种抽象，它只认识 DOM，因此，React 会先将 JSX 转换为 React element，这是一个简单结构化对象，在 `src\isomorphic\classic\element\ReactElement.js` 中，我们可以看到，创建 React element 的过程如下：

    1. 校验并设置组件的 `ref` 和 `key`。
    2. 设置 element 的子孙 `props.children`。
    3. 合并 `defaultProps` 到 `props`，用默认属性替换掉未定义属性。
    4. 设置 element 类型 `props.$$typeof`，服务于类型判断和 debug

Demo 中的  `<ExampleApplication />` 将会被转为下面这个 React element：

```js
{
   $$typeof: Symbol(react.element),
   key: null,
   props: {
     hello: 'world'
   },
   ref: null,
   type: function ExampleApplication(props) {},
   // .....
}
```

### ReactMount

实际上，`ReactDOM.render` 中并不存在任何逻辑，它引用了 `ReactMount.render` 进行渲染，我们看到 `src/renderers/dom/client/ReactMount.js` 开头的注释：

```js
/**
 * Mounting is the process of initializing a React component by creating its
 * representative DOM elements and inserting them into a supplied `container`.
 * Any prior content inside `container` is destroyed in the process.
 *
 *   ReactMount.render(
 *     component,
 *     document.getElementById('container')
 *   );
 *
 *   <div id="container">                   <-- Supplied `container`.
 *     <div data-reactid=".3">              <-- Rendered reactRoot of React
 *       // ...                                 component.
 *     </div>
 *   </div>
 *
 * Inside of `container`, the first element rendered is the "reactRoot".
 */
var ReactMount = {
  TopLevelWrapper: TopLevelWrapper,
  // ......
}
```

React 对挂载的定义是：通过创建组件对应的 DOM 元素并将其插入到指定的 `container` ，来初始化一个 React 组件，下面这个代码展示了一个插入到 `div.container` 中的 React 组件：

```html
<div id="container">
  <div data-reactid=".3">
  <!-- ... -->
  </div>
</div>
```

JSX 被转换为 React element 后，依据不同的 element 类型，初始化不同的 **React Internal Component**，它们即是 React 中的 Virtual Dom：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/0/jsx-to-vdom.svg)

在 `ReactMount.render()` 方法中，首先会为 `<ExampleApplication />` 对应的 React element 做一个**顶层包裹（TopLevelWrapper）**：

```js
// src/renderers/dom/client/ReactMount.js#470
var nextWrappedElement = React.createElement(TopLevelWrapper, {
  child: nextElement,
});
```

`TopLevelWrapper` 提供了一个内容不多的 `render()` 方法，React 将从 `TopLevelWrapper` 开始渲染组件树，`TopLevelWrapper` 的 render 方法之后会返回 `<ExampleApplication />` 组件：

```js
// src/renderers/dom/client/ReactMount.js#281
TopLevelWrapper.prototype.render = function() {
  return this.props.child;
};
```

之后，根据不同的 element 类型，初始化对应的 React component：

```js
// src/renderers/dom/client/ReactMount.js#385
var componentInstance = instantiateReactComponent(nextElement, false);
```

`<ExampleApplication />` 对应的 React component 如下：

```js
{
  context: {},
  props: {hello: 'word'},
  refs: {},
  state: {message: 'no message'},
  updater: {
    isMounted: function() {},
    // ...
  }
  // ...
}
```

## 总结

Part-0 的内容可以概括如下：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/0/part-0-B.svg)

实质：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/0/part-0-C.svg)

## 补充

在 `src\isomorphic\classic\element\ReactElement.js` 能学到：

1. 保存常用方法的引用，以优化属性检索速度：

```js
var hasOwnProperty = Object.prototype.hasOwnProperty

// ....
function hasValidKey(config) {
  if (__DEV__) {
    if (hasOwnProperty.call(config, 'key')) {
      var getter = Object.getOwnPropertyDescriptor(config, 'key').get;
      if (getter && getter.isReactWarning) {
        return false;
      }
    }
  }
  return config.key !== undefined;
}
```

2. 使用 [`Object.defineProperty()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) 来更加细粒度地控制对象属性：

```js
Object.defineProperty(element, '_source', {
  configurable: false,
  enumerable: false,
  writable: false,
  value: source,
})
```

3. 使用 [`Object.freeze()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze) 来冻结对象属性，防止向对象增删属性，防止对象属性及属性描述（可迭代性、可配置性、可写性等）被更改：

```js
if (Object.freeze) {
  Object.freeze(element.props);
  Object.freeze(element);
}
```

4. React 组件生命期：

```js
/**
 * ------------------ The Life-Cycle of a Composite Component ------------------
 *
 * - constructor: Initialization of state. The instance is now retained.
 *   - componentWillMount
 *   - render
 *   - [children's constructors]
 *     - [children's componentWillMount and render]
 *     - [children's componentDidMount]
 *     - componentDidMount
 *
 *       Update Phases:
 *       - componentWillReceiveProps (only called if parent updated)
 *       - shouldComponentUpdate
 *         - componentWillUpdate
 *           - render
 *           - [children's constructors or receive props phases]
 *         - componentDidUpdate
 *
 *     - componentWillUnmount
 *     - [children's componentWillUnmount]
 *   - [children destroyed]
 * - (destroyed): The instance is now blank, released by React and ready for GC.
 *
 * -----------------------------------------------------------------------------
 */
```