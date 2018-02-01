# Part 14

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/14/part-14.svg)

## 内容

### 子孙的更新

在我们的例子中，`<ExampleApplication>` 组件需要渲染的内容为 ：

```html
<div>
  <button onClick={this.onClickHandler.bind(this)}>set state button</button>
  <ChildCmp childMessage={this.state.message} />
  And some text as well!
</div>
```

可见，其第一个子孙为一个 `<div>`，在 `ReactDOMComponent._updateDOMChildren()` 方法中，会针对节点子孙的复杂性进行不同的更新策略。这里复杂性可以概括为：

>  内容只是字符串、数字或者 HTML 文本的节点算是简单节点

```js
// src/renderers/dom/shared/ReactDOMComponent.js#1080
var lastContent = CONTENT_TYPES[typeof lastProps.children]
      ? lastProps.children
      : null;
var nextContent = CONTENT_TYPES[typeof nextProps.children]
	? nextProps.children
	: null;

var lastHtml =
    lastProps.dangerouslySetInnerHTML &&
    lastProps.dangerouslySetInnerHTML.__html;
var nextHtml =
    nextProps.dangerouslySetInnerHTML &&
    nextProps.dangerouslySetInnerHTML.__html;

// Note the use of `!=` which checks for null or undefined.
var lastChildren = lastContent != null ? null : lastProps.children;
var nextChildren = nextContent != null ? null : nextProps.children;
```

### 更新简单节点

如果内容是简单节点，如文本或者是 HTML，本例中 `<ChildCmp>` 下的  则通过 `ReactMultiChild.updateTextContent()` 或者 `ReactMultiChild.updateMarkup()` 进行节点更新:

```JS
// src/renderers/dom/shared/ReactDOMComponent.js#1113
this.updateTextContent('' + nextContent);
```

值得注意的是，每个更新都是一个结构体对象：

```js
// src/renderers/shared/stack/reconciler/ReactMultiChild.js#319
var updates = [makeTextContent(nextContent)];
/* 
updates = [{
  type: 'TEXT_CONTENT',
  content: textContent,
  fromIndex: null,
  fromNode: null,
  toIndex: null,
  afterNode: null,
}]
*/
processQueue(this, updates);
```

在 `DOMChildrenOperations` 中，针对不同的更新类型，将作出不同的更新策略：

```js
//src/renderers/dom/client/utils/DOMChildrenOperations.js#172
processUpdates: function(parentNode, updates) {
    for (var k = 0; k < updates.length; k++) {
      var update = updates[k];

      switch (update.type) {
        case 'INSERT_MARKUP':
          insertLazyTreeChildAt(
            parentNode,
            update.content,
            getNodeAfter(parentNode, update.afterNode)
          );
          break;
        case 'MOVE_EXISTING':
          moveChild(
            parentNode,
            update.fromNode,
            getNodeAfter(parentNode, update.afterNode)
          );
          break;
        case 'SET_MARKUP':
          setInnerHTML(
            parentNode,
            update.content
          );
          break;
        case 'TEXT_CONTENT':
          setTextContent(
            parentNode,
            update.content
          );
          break;
        case 'REMOVE_NODE':
          removeChild(parentNode, update.fromNode);
          break;
      }
    }
  }
```



### 更新复杂节点

看到这个 `<div>` 的内容，显然其不是一个 **“简单”** 节点。因此，会调用 `ReactMultiChild.updateChildren()` 进行子孙的更新，其内部调用的是 `ReactMultiChild._reconcilerUpdateChildren()` 进行的更新：

```js
// src/renderers/dom/shared/ReactDOMComponent.js#1104
if (lastChildren != null && nextChildren == null) {
  this.updateChildren(null, transaction, context);
}
```

而在 `ReactMultiChild.__reconcilerUpdateChildren()` 方法内部，又是调用的 `ReactChildReconciler.updateChildren()` 进行的更新，其更新过程如下：

1. 通过 `shouldUpdateReactComponent` 判断组件是否需要更新，若需要更新，则通过 `ReactReconciler.receiveComponent()` 进行更新。

2. 如果不需要更新，判断是否需要通过 `ReactReconciler.unmountComponet()` 进行组件卸载。并且将其替换为新的组件，即挂载新的组件。

3. 重复上面的步骤，直到子孙节点内容为简单内容。

   ![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/14/children-update.svg)



## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/14/part-14-B.svg)

本质上:

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/14/part-14-C.svg)

现在，整个更新过程也可以概括为：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/14/updating-parts-C.svg)