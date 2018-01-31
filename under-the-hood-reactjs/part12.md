# Part 12

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/12/part-12.svg)

## 内容

### `shouldUpdateReactComponent`

上一部分中，我们知道，借由 `shouldUpdateReactComponent` 模块，可以判断当前组件实例是应当更新？销毁？还是重新创建一个新的实例，接下来我们分析其执行流程。

```js
// src/renderers/shared/shared/shouldUpdateReactComponent.js#25
function shouldUpdateReactComponent(prevElement, nextElement) {
    var prevEmpty = prevElement === null || prevElement === false;
    var nextEmpty = nextElement === null || nextElement === false;
    if (prevEmpty || nextEmpty) {
        return prevEmpty === nextEmpty;
    }

    var prevType = typeof prevElement;
    var nextType = typeof nextElement;
    if (prevType === 'string' || prevType === 'number') {
        return (nextType === 'string' || nextType === 'number');
    } else {
        return (
            nextType === 'object' &&
            prevElement.type === nextElement.type &&
            prevElement.key === nextElement.key
        );
    }
}
```

在 React 官方文档的 [Reconciliation 一章](https://reactjs.org/docs/reconciliation.html) 中，我们知道，React 这么做的动机是想最小化 DOM 树变动的耗时，这成为了影响 React 性能的关键一步。

DOM 的差异比较步骤可以概括为：

1. 首先比较渲染前后的两个 element。如果两个 element 的类型不一致，或者指定的 `key` 不一致：

   ```js
   prevElement.type !== nextElement.type || prevElement.key === nextElement.key
   ```

   例如 `<a>` 与 `<div>`、`<Article>` 与 `<Comment>` 或者 `<a>` 与 `<Article>` 。那么 React 就直接砍掉老的 DOM 树（unmount），从零构建新的树（mount）。下面这个变动会卸载掉老的 `<Counter>` ，重新挂载新的 `<Counter>`：

   ```html
   <div>
     <Counter />
   </div>

   <span>
     <Counter />
   </span>
   ```

2. 如果渲染前后的两个 element 为同类型的 **DOM element**，React 只修改 DOM 节点变更的 attributes 和 property，以及修改变更的样式。

   ```html
   <div className="before" title="stuff" />

   <div className="after" title="stuff" />

   <div style={{color: 'red', fontWeight: 'bold'}} />

   <div style={{color: 'green', fontWeight: 'bold'}} />
   ```


1. 如果渲染前后的两个 element 为同类型的 **Component element**，组件实例所声明的 `componentWillReceiveProps()` 和 `componentWillUpdate()` 将被调用，接下来，将调用组件的 `render()` 方法开始递归地渲染其子孙。

## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/12/part-12-B.svg)

本质上：

![](https://camo.githubusercontent.com/1b60c563ec6c281f355f48187b442c59470eb549/68747470733a2f2f7261776769742e636f6d2f426f6764616e2d4c79617368656e6b6f2f556e6465722d7468652d686f6f642d52656163744a532f6d61737465722f737461636b2f696d616765732f31322f706172742d31322d432e737667)



