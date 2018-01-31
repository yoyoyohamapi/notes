# Part 4

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/4/part-4.svg)

## 内容

### `ReactDomComponent.mountComponent()`

前面内容，我们了解了 `ReactCompositeComponent` 的挂载方法 `mountComponent()` 方法，挂载了 `<ExampleApplication />` 之后，也该挂载它的 `<div>` 子孙了。 这一部分则会讨论 `ReactDomComponent` 的 `mountComponent()` 方法：

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
}
```

可以看到，对于一些复杂的 tag，如 `<audio />`，就需要通过 `trapBubbledEventsLocal()` 方法，在组件挂载完成后，为其绑定事件监听，例如会为 `<audio />`、`<video />` 等多媒体 tag 绑定上诸如 play、pause 这样的多媒体事件。

```js
function trapBubbledEventsLocal() {

  switch (inst._tag) {
    case 'iframe':
    case 'object':
      inst._wrapperState.listeners = [
        ReactBrowserEventEmitter.trapBubbledEvent('topLoad', 'load', node),
      ];
      break;
    case 'video':
    case 'audio':
      inst._wrapperState.listeners = [];
      // Create listener for each media event
      for (var event in mediaEvents) {
        if (mediaEvents.hasOwnProperty(event)) {
          inst._wrapperState.listeners.push(
            ReactBrowserEventEmitter.trapBubbledEvent(
              event,
              mediaEvents[event],
              node,
            ),
          );
        }
      }
      break;
    // ....
    case 'input':
    case 'select':
    case 'textarea':
      inst._wrapperState.listeners = [
        ReactBrowserEventEmitter.trapBubbledEvent(
          'topInvalid',
          'invalid',
          node,
        ),
      ];
      break;
  }
}
```

而对于 `<input />` 等组件，还会有对应的模块来进行挂载时的 wrapper 操作，例如 `ReactDOMInput` 就在 `<input />` 挂载时初始化了一些状态：

```js
//  src/renderers/dom/client/wrappers/ReactDOMInput.js
var ReactDOMInput = {
  // ...
  mountWrapper: function(inst, props) {
    // ....
    var defaultValue = props.defaultValue;
    inst._wrapperState = {
      initialChecked: props.checked != null
        ? props.checked
        : props.defaultChecked,
      initialValue: props.value != null ? props.value : defaultValue,
      listeners: null,
      onChange: _handleChange.bind(inst),
      controlled: isControlled(props),
    };
  },

  // ...
}
```

### 属性校验

之后，`ReactDOMComponent.mountComponent()` 将会进行属性校验：

```js
function assertValidProps(component, props) {
  if (!props) {
    return;
  }
  // ...
  if (props.dangerouslySetInnerHTML != null) {
    invariant(
      props.children == null,
      'Can only set one of `children` or `props.dangerouslySetInnerHTML`.',
    );
    invariant(
      typeof props.dangerouslySetInnerHTML === 'object' &&
        HTML in props.dangerouslySetInnerHTML,
      '`props.dangerouslySetInnerHTML` must be in the form `{__html: ...}`. ' +
        'Please visit https://fb.me/react-invariant-dangerously-set-inner-html ' +
        'for more information.',
    );
  }
  // ...
  invariant(
    props.style == null || typeof props.style === 'object',
    'The `style` prop expects a mapping from style properties to values, ' +
      "not a string. For example, style={{marginRight: spacing + 'em'}} when " +
      'using JSX.%s',
    getDeclarationErrorAddendum(component),
  );
}
```

例如，校验 `props.dangerouslySetInnerHTML` 是否含有 `__html` 属性，校验 `props.style` 是否是一个对象等。


### 创建 HTML
最终，将通过 `document.createElement()` 来创建真正的 html element。

## 总结

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/4/part-4-B.svg)

实质：

![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/4/part-4-C.svg)

