# Web Component

已经有不少流行的第三方库用来支持前端组件化开发了，比如 Angular、React 以及 Vue。现在，浏览器原生也提供了组件化开发的能力，这都依赖于 `customElements` 这个全局对象，它提供了三个 API：

- `define(tagName, constructor, options)`：用来创建一个 HTML 元素。
- `get(tagName)`：用来获取定义好的 HTML 元素。
- `whenDefined(tagName)`：这个 API 将返回一个 Promise 对象。当元素定义完成后，promise 会被 resolved，如果 tag name 无效，则会被 rejected。

## 自定义 HTML 元素

```js
class MyCustomElement extends HTMLElement {
  constructor() {
    super();
    // …
  }

  // …
}

customElements.define('my-custom-element', MyCustomElement);
```

当然，如果不想污染当前作用域，也可以直接使用匿名类：

```js
customElements.define('my-custom-element', class extends HTMLElement {
  constructor() {
    super();
    // …
  }

  // …
});
```

## 自定义元素解决了什么问题？

如果没有自定义元素，原生的 HTML 元素不够语义化，我们只能不停地嵌套 div：

```html
<div class="top-container">
  <div class="middle-container">
    <div class="inside-container">
      <div class="inside-inside-container">
        <div class="are-we-really-doing-this">
          <div class="mariana-trench">
            …
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

例如，要实现下面这个菜单栏：

![img](https://cdn-images-1.medium.com/max/800/0*v56OyrPtg_cZzzaZ)

我们将要书写的 HTML 会是：

```html
<div class="primary-toolbar toolbar">
  <div class="toolbar">
    <div class="toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-undo">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
    <div class="toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-redo">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
    <div class="toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-print">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
    <div class="toolbar-toggle-button toolbar-button">
      <div class="toolbar-button-outer-box">
        <div class="toolbar-button-inner-box">
          <div class="icon">
            <div class="icon-paint-format">&nbsp;</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

这段 HTML 代码一方面层级很深，一方面不够语义化，开发者只能通过 css class name 了解组件含义。

如果，它能够是下面这样的，是不是就清晰太多：

```html
<primary-toolbar>
  <toolbar-group>
    <toolbar-button class="icon-undo"></toolbar-button>
    <toolbar-button class="icon-redo"></toolbar-button>
    <toolbar-button class="icon-print"></toolbar-button>
    <toolbar-toggle-button class="icon-paint-format"></toolbar-toggle-button>
  </toolbar-group>
</primary-toolbar>
```

## 重用

假定我们有下面的 HTML 结构：

```html
<div class="my-custom-element">
  <input type="text" class="email" />
  <button class="submit"></button>
</div>
```

假定我想复用这个结构，理想的方式，就是把它封装成一个组件：

```html
<my-custom-element></my-custom-element>
```

并且，不要忘了指定组件逻辑，现代 Web 应用中，很少再有纯静态的组件了：

```js
const myDiv = document.querySelector('.my-custom-element');

myDiv.addEventListener('click', _ => {
  myDiv.innerHTML = '<b> I have been clicked </b>';
});
```

但是，更好地做法是，将组件逻辑封装到组件内部：

```js
class MyCustomElement extends HTMLElement {
  constructor() {
    super();

    this.addEventListener('click', _ => {
      this.innerHTML = '<b> I have been clicked </b>';
    });
  }
}

customElements.define('my-custom-element', MyCustomElement);
```

## 约定

在创建自定义 HTML 元素时，需要尊属下面这些规则：

- 元素名称必须包含 `-`，`<myCustomElement>` 和 `<my_custom_element>` 是不允许的。
- 不允许创建同名的元素，如果你这么做了，浏览器会抛出 `DOMException` 异常。
- 自定义的元素无法 self-closing。即我们不能 `<my-custom-element />`。

## 能力

自定义元素可以做许许多多的事儿，首先就是将组件逻辑内聚到组件内部：

```js
class MyCustomElement extends HTMLElement {
  // ...

  constructor() {
    super();

    this.addEventListener('mouseover', _ => {
      console.log('I have been hovered');
    });
  }

  // ...
}
```

这其中，我们可以利用到许多钩子：

`constructor`

在这个 hook 中，可以做：状态初始化，注册事件监听，创建 shadow DOM 等事儿。一定要记得调用 `super()`。

`connectedCallback`

每当元素被添加到 DOM 中时，这个 hook 会被响应。也就是，如果我们想在元素渲染后，再执行一些工作时，例如 AJAX 请求数据等，可以关注这个 hook。

`disconnectedCallback`

每当元素被移出 DOM 时，这个 hook 会被响应，此时我们可以做一些资源清理的工作。但要知道的是，如果用户关闭了 tab 页，这个 hook 不会被响应。

`attributeChangedCallback`

如果元素的属性（只针对白名单内的元素）发生了变化，例如删除，增加，修改等，该 hook 会被响应。当元素被 parser 创建时，该 hook 也会被调用。

`addoptedCallback`

该 hook 会在 `document.adoptNode()` 时执行，即当元素要被放到不同的文档时。

> 上述所有的 hook 都是同步的。

## 属性反射

对于原生的 HTML 元素来说，如果我们在 JavaScript 中设置了其属性：

```js
myDiv.id = 'new-id'
```

其属性能够被自动映射到 HTML 上：

```html
<div id="new-id">...</div>
```

然而，自定义元素就无法做到，我们需要通过 getter 及 setter 完成类似功能：

```js
class MyCustomElement extends HTMLElement {
  // ...

  get myProperty() {
    return this.hasAttribute('my-property');
  }

  set myProperty(newValue) {
    if (newValue) {
      this.setAttribute('my-property', newValue);
    } else {
      this.removeAttribute('my-property');
    }
  }

  // ...
}
```

## 继承与扩展

借助于 class，我们还可以通过 extends 实现自定义元素的扩展：

```js
class MyAwesomeButton extends MyButton {
  // ...
}

customElements.define('my-awesome-button', MyAwesomeButton)
```

如果是要继承内置元素，我们还需要传递第三个参数，声明我们想要从哪个内置元素进行继承，因为有许多不同的内置元素继承自同一个类：

```js
class MyButton extends HTMLButtonElement {
  // ...
}

customElements.define('my-button', MyButton, {extends: 'button'})
```

> 需要注意的是，继承内置元素现在只有 Chrome 67+ 支持。

## 更新元素

我们使用 `customeElements.define(...)` 完成了元素的注册。如果我们想要知道某个元素完成了注册，就可以使用下面的方式：

```js
customElements.whenDefined('my-custom-element').then(_ => {
  console.log('My custom element is defined');
});
```

这是因为元素的注册可以延后，比如我们需要顺序注册一些元素时，我们可以先通过 `whenDefined` 声明好该元素注册后的行为。

## shadow DOM

自定义元素和 shadow DOM 最好配合使用。前者用于组件化开发，将组件逻辑进行内聚。后者则是为组件中的 DOM 创建了一个隔离的环境。

为了在自定义元素中使用 shadow DOM，你可以使用 `this.attachShadow` 将 shadow DOM 绑定到当前元素：

```js
class MyCustomElement extends HTMLElement {
  // ...
  
  constructor() {
    super();
    
    const shadowRoot = this.attachShadow({mode: 'open'})
    const elementContent = document.createElement('div')
    
    shadowRoot.appendChild(elementContent)
  }
}
```

## 模板

有了 shadow DOM，我们还可以使用模板来解耦 shadow DOM 中的 UI 到模板，从而分离 UI 和 JavaScript 逻辑：

```html
<template id="my-custom-element-template">
  <div class="my-custom-element">
    <input type="text" class="email" />
    <button class="submit"></button>
  </div>
</template>
```

```js
const myCustomElementTemplate = document.querySelector('#my-custom-element-template')

class MyCustomElement extends HTMLElement {
  // ...
  
  constructor() {
    super()
    
    const shadowRoot = this.attachShadow({mode: 'open'})
    shadowRoot.appendChild(myCustomElementTemplate.content.cloneNode(true))
  }
}
```

## 样式

声明自定义元素样式的方式和声明一般元素的样式一致：

```css
my-custom-element {
  border-radius: 5px;
  width: 30%;
  height: 50%;
  // ...
}
```

有时候，我们希望在元素定义（注册）后再声明样式，这可以这么做：

```css
my-button:not(:defined) {
  height: 20px;
  width: 50px;
  opacity: 0;
}
```

## Unknown elements 与 undefined custom elements

如果浏览器未能识别标签，那么它会被解析为 `HTMLUnknownElement`：

```js
const element = document.createElement('thisElementIsUnknown')

if (element instanceof HTMLUnknownElement) {
  console.log('The selected element is unknown')
}
```

但是，这对自定义元素并不适用，如果某个元素没有被定义，则会被浏览器解析为 `HTMLElement` ，并被当做一个未定义（undefined）元素：

```js
const element = document.createElement('this-element-is-undefined');

if (element instanceof HTMLElement) {
  console.log('The selected element is undefined but not unknown');
}
```

## 浏览器支持

通过下面的手段可以探测浏览器是否支持自定义元素，如果不支持，就考虑使用 polyfill：

```js
function loadScript(src) {
  return new Promise(function(resolve, reject) {
    const script = document.createElement('script');

    script.src = src;
    script.onload = resolve;
    script.onerror = reject;

    document.head.appendChild(script);
  });
}

const supportsCustomElements = 'customElements' in window;

if (supportsCustomElements) {
  // Browser supports custom elements natively. You're good to go.
} else {
  loadScript('path/to/custom-elements.min.js').then(_ => {
    // Custom elements polyfill loaded. You're good to go.
  });
}
```

## 总结

Web Component 能给你下面这些能力：

- 封装 JavaScript 逻辑及 CSS 样式到一个新的 HTML 元素
- 允许你扩展现有的 HTML 元素
- 不需要引入第三方库
- 可以与其他特性无缝衔接（如 shadow DOM， templates， slots 等等）
- 适配浏览器的 dev tools
- 能够利用现有的可访问性特性