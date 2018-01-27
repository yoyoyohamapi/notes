# Part 6

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/6/part-6.svg)

## 内容

### `DOMLazyTree` 模块

分析完 `ReactDOMComponent` 挂载组件时的属性比较后，我们在 `mountComponent()` 看到了，会用 `DomLazyTree` 来对得到的 DOM 节点进行包裹：

```js
// src/renderers/dom/shared/ReactDOMComponent.js#627
var lazyTree = DOMLazyTree(el);
```

在 `DOMLazyTree` 模块的注释可以看到：在 IE 系列浏览器（IE 8 - Edge）中，添加**没有子孙的 DOM 节点**要远远快于添加**一颗完整的子 DOM 树**。因此，`DOMLazyTree` 会检测我们的应用环境，如果是 IE，那么我们在添加某个 DOM 节点到另一 DOM 节点下之前，会暂时先不为这个待添加节点填充子孙，而会在待添加节点加入到父节点后，再为其添加子孙。这个性能佐证可以参看 ![https://github.com/sophiebits/innerhtml-vs-createelement-vs-clonenode](https://github.com/sophiebits/innerhtml-vs-createelement-vs-clonenode)。


```js
// src/renderers/dom/client/utils/DOMLazyTree.js#87
function queueChild(parentTree, childTree) {
  if (enableLazy) {
    parentTree.children.push(childTree);
  } else {
    parentTree.node.appendChild(childTree.node);
  }
}
```

### 挂载子孙

接下来，`ReactDOMComponent.mountComponent()` 将会进行组件子孙的创建：

```js
// src/renderers/dom/shared/ReactDOMComponent.js#628
this._createInitialChildren(transaction, props, context, lazyTree);
```

`_.createInitialChildren()` 主要完成了：

1. 挂载子孙：

```js
// src/renderers/dom/shared/ReactDOMComponent.js#841
var mountImages = this.mountChildren();
```

这里的 `mountImages` 就是我们的 DOM node 列表，在 demo 中，就是 `[<button />, <div />, //...]`。

2. 将子孙添加到 DOM 树中，会针对 IE 做特殊处理：

```js
// src/renderers/dom/shared/ReactDOMComponent.js#846
for (var i = 0; i < mountImages.length; i++) {
  DOMLazyTree.queueChild(lazyTree, mountImages[i]);
}
```

### 挂载过程的回顾

通过下面这个图再次回顾下挂载过程：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/6/overall-mounting-scheme.svg)

1. React 将我们自定义的组件（`<ExampleApplication />`）实例化为 `ReactCompositeComponent` 对象，并且挂载了它 （`ReactCompositeComponent.mountComponent()`）。

2. 挂载伊始，会借由构造函数创建自定义组件的实例。

3. 之后，调用组件的 `render()` 方法，`React.createElement` 将创建 React element。

4. 我们遇到了 `div` 类型的 element，就通过 `ReactDOMComponent` 来实例化 `ReactDOMComponent` 对象。

5. 之后，我们需要挂载 DOM 组件（`ReactDOMComponent.mountComponent()`），创建 DOM 节点并赋给其属性及事件监听器等。

6. 再然后，我们处理了 DOM 组件的子孙。我们创建了子孙的实例，并且将它们也给挂载，根据子孙不同的类型，挂载过程会跳到步骤 1 或者步骤 5。

## 总结

![](https://bogdan-lyashenko.github.io/Under-the-hood-ReactJS/stack/book/Part-6.html)

实质上：

![](https://bogdan-lyashenko.github.io/Under-the-hood-ReactJS/stack/book/Part-6.html)