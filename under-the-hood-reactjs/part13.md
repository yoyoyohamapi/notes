# Part 13

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/13/part-13.svg)

## 内容

### Receive Component

当确定我们的 DOM element 需要更新时，`ReactReconciler.receiveComponent()` 将调用 `ReactDOMComponent.receiveComponent()` 来更新组件，主要完成：（1）更新 DOM 节点的属性（2）更新 DOM 节点的子孙：

```js
// src/renderers/dom/shared/ReactDOMComponent.js#901
this._updateDOMProperties(lastProps, nextProps, transaction);
this._updateDOMChildren(lastProps, nextProps, transaction, context);
```

## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/13/part-13-B.svg)



本质上：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/13/part-13-C.svg)