# Part 5

## 内容

!()[https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/5/part-5.svg]

### DOM 属性更新

在 React DOM Component 挂载的时候，还会更新 DOM 属性：

```js
// src/renderers/dom/shared/ReactDOMComponent.js#486
mountComponent: function(
    transaction,
    hostParent,
    hostContainerInfo,
    context,
  ) {
    // ...
    switch (this._tag) {
      case 'audio':
      case 'form':
      case 'iframe':
      case 'img':
      case 'link':
      case 'object':
      case 'source':
      case 'video':
        this._wrapperState = {
          listeners: null,
        };
        transaction.getReactMountReady().enqueue(trapBubbledEventsLocal, this);
        break;
      case 'input':
        ReactDOMInput.mountWrapper(this, props, hostParent);
        props = ReactDOMInput.getHostProps(this, props);
        transaction.getReactMountReady().enqueue(trackInputValue, this);
        transaction.getReactMountReady().enqueue(trapBubbledEventsLocal, this);
        break;
      // ....
    }

    assertValidProps(this, props);

    // ...

    el = ownerDocument.createElement()

    this._updateDOMProperties(null, props, transaction)
}
```
在 `_.updateDOMProperties` 的函数注释中我们知道，通过检测属性间的差异以及更新 DOM 是 Reconcile 算法的必需。该函数式性能优化的关键步骤。

### `lastProps` 循环

在 `_.updateDOMProperties` 中，将会进行对 `lastProps`（前一次属性） 和 `nextProps`（下一次属性） 的两个循环，以比较属性变更，不做不必要的更新。

在对上一次属性循环时，会考虑如下几个问题：

- 会考虑样式（style）属性，因为样式重绘的开销比较大，从源码中我们看到，只会对改变的样式更新，如果样式不发生改变，则接下来的属性 `nextProps` 会沿用样式的拷贝。

- 删除事件绑定
- 删除 DOM 节点的 attribute 绑定
- 删除 DOM 节点的属性

```js
_updateDOMProperties: function(lastProps, nextProps, transaction) {
  // ...
  for (propKey in lastProps) {
    // ....
    if (propKey === STYLE) {
      var lastStyle = this._previousStyleCopy;
      for (styleName in lastStyle) {
        if (lastStyle.hasOwnProperty(styleName)) {
          styleUpdates = styleUpdates || {};
          styleUpdates[styleName] = '';
        }
      }
      this._previousStyleCopy = null;
      } if (propKey === STYLE) {
        var lastStyle = this._previousStyleCopy;
        for (styleName in lastStyle) {
          if (lastStyle.hasOwnProperty(styleName)) {
            styleUpdates = styleUpdates || {};
            styleUpdates[styleName] = '';
          }
        }
        this._previousStyleCopy = null;
      } else if (registrationNameModules.hasOwnProperty(propKey)) {
        if (lastProps[propKey]) {
          // Only call deleteListener if there was a listener previously or
          // else willDeleteListener gets called when there wasn't actually a
          // listener (e.g., onClick={null})
          deleteListener(this, propKey);
        }
      } else if (isCustomComponent(this._tag, lastProps)) {
        if (!RESERVED_PROPS.hasOwnProperty(propKey)) {
          DOMPropertyOperations.deleteValueForAttribute(getNode(this), propKey);
        }
      } else if (
        DOMProperty.properties[propKey] ||
        DOMProperty.isCustomAttribute(propKey)
      ) {
        DOMPropertyOperations.deleteValueForProperty(getNode(this), propKey);
      }
    }
  }
} 
```

### `nextProps` 循环

而在对 `nextProps` 循环中，则会进行：

- 对于样式属性的更新，只更新相较于 `lastProp` 发生变化的样式
- 对于下一次属性中声明的事件，将事件绑定过程入队
- 删除不再需要的事件监听
- 设置 DOM node 的 attributes
- 设置 DOM node 的属性，删除空（`null`、`undefined`）的属性


```js
_updateDOMProperties: function(lastProps, nextProps, transaction) {
  for (propKey in nextProps) {
    var nextProp = nextProps[propKey];
    var lastProp = propKey === STYLE
      ? this._previousStyleCopy
      : lastProps != null ? lastProps[propKey] : undefined;
    if (
      !nextProps.hasOwnProperty(propKey) ||
      nextProp === lastProp ||
      (nextProp == null && lastProp == null)
    ) {
      continue;
    }
    if (propKey === STYLE) {
      if (nextProp) {
        nextProp = this._previousStyleCopy = Object.assign({}, nextProp);
      } else {
        this._previousStyleCopy = null;
      }
      if (lastProp) {
        // Unset styles on `lastProp` but not on `nextProp`.
        for (styleName in lastProp) {
          if (
            lastProp.hasOwnProperty(styleName) &&
            (!nextProp || !nextProp.hasOwnProperty(styleName))
          ) {
            styleUpdates = styleUpdates || {};
            styleUpdates[styleName] = '';
          }
        }
        // Update styles that changed since `lastProp`.
        for (styleName in nextProp) {
          if (
            nextProp.hasOwnProperty(styleName) &&
            lastProp[styleName] !== nextProp[styleName]
          ) {
            styleUpdates = styleUpdates || {};
            styleUpdates[styleName] = nextProp[styleName];
          }
        }
      } else {
        // Relies on `updateStylesByID` not mutating `styleUpdates`.
        styleUpdates = nextProp;
      }
    } else if (registrationNameModules.hasOwnProperty(propKey)) {
      if (nextProp) {
        enqueuePutListener(this, propKey, nextProp, transaction);
      } else if (lastProp) {
        deleteListener(this, propKey);
      }
    } else if (isCustomComponent(this._tag, nextProps)) {
      if (!RESERVED_PROPS.hasOwnProperty(propKey)) {
        DOMPropertyOperations.setValueForAttribute(
          getNode(this),
          propKey,
          nextProp,
        );
      }
    } else if (
      DOMProperty.properties[propKey] ||
      DOMProperty.isCustomAttribute(propKey)
    ) {
      var node = getNode(this);
      if (nextProp != null) {
        DOMPropertyOperations.setValueForProperty(node, propKey, nextProp);
      } else {
        DOMPropertyOperations.deleteValueForProperty(node, propKey);
      }
    }
  }
}
```

## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/5/part-5-B.svg)

实质上：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/5/part-5-C.svg)