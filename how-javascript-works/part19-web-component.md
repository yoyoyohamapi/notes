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



