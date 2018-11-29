# Shadow DOM

Shadow DOM 是 Web Components 的实现之一，其特点有：

- **隔离的 DOM**: 组件的 DOM 是自包含的（self-contained），即，诸如 `document.querySelector()` 这样的 API 不会返回 shadow DOM 中的节点。shadow DOM 是相互隔离的，这也极大的简化了 web 应用中 CSS 选择器的使用，我们不用担心 class 或者 id 的名字冲突。
- **Scoped CSS**：定义在 shadow DOM 中的 CSS 是 scoped 的，不用担心污染外部的样式。
- **Composition**：能够为你的组件设计声明式的，基于标记的 API。

Shadow DOM 与一般的 DOM 的区别在于：

- 如何在页面中创建及使用 shadow DOM
- 如何与页面其余部分交互

对于普通 DOM 来说，你首先创建 DOM 节点，然后将节点作为 children 追加到其他 DOM 节点下 。而对于 shadow   DOM，你创建了一棵 scoped DOM tree 绑定到元素上，但这棵树实际上独立于其 children。这棵树被称作 **shadow tree**，**shadow tree** 绑定到的元素（element）则被称为 **shadow host**。任何 shadow tree 的节点都局部于（local） host 元素，包括样式定义 `<style>`，这也就是 CSS 样式也能够进行 scoped 的原因。

## 创建 Shadow DOM

**shadow root** 是一个 document fragment，它被绑定到一个 host 元素（宿主元素）。当你把 shadow root 绑定到元素上，这个元素也就获得了它的 shadow DOM。

为一个元素创建 shadow DOM 的方式为：`element.attachShadow()`：

```js
// 创建宿主元素
const header = document.createElement('header')
// 为宿主元素创建 shadow dom，并获得 shadow dom 的根节点
const shadowRoot = header.attachShadow({mode: 'open'})
const paragraphElement = document.createElement('p')

paragraphElement.innerText = 'Shadow DOM'
// 从根节点开始，设置 shadow DOM 的内容
shadowRoot.appendChild(paragraphElemenowRoot.appendChild(paragraphElement)
```

## Light DOM

Light DOM 是组件使用者所创建的标记，该 DOM 存在于组件的 shadow dom 之外，其实际上是 HTML 元素的 children。假定我们基于原生 button 扩展了一个 extend-button，并在在其中插入了图像和文本：

```html
<extended-button>
  <!-- the image and span are extended-button's light DOM -->
  <img src="boot.png" slot="image">
  <span>Launch</span>
</extended-button>
```

这里，图像和文本块就是 light DOM， `<extended-button>` 则是 **shadow DOM**。shadow DOM 是局部于组件的，定义了组件内部结构，scoped css 以及实现细节。

## 扁平化的 DOM 树

浏览器会将用户所创建的 light DOM 放到 shadow DOM 中，渲染得到最终结果。你将在 DevTools 中看到这个渲染结果，它是一棵扁平化后的树，从 shadow root 开始，其内容一览无遗：

```html
<extended-button>
  #shadow-root
  <style>…</style>
  <slot name="image">
    <img src="boot.png" slot="image">
  </slot>
  <span id="container">
    <slot>
      <span>Launch</span>
    </slot>
  </span>
</extended-button>
```

## 模板

如果页面中有重复的 UI 结构时，理想的做法是将它抽象成模板：

```html
<template id="my-paragraph">
  <p> Paragraph content. </p>
</template>
```

模板不会立即渲染到页面，除非你通过 JavaScript 获得其内容，并将内容注入到现有的 DOM 结构：

```js
const template = document.getElementById('my-paragraph');
const templateContent = template.content;
document.body.appendChild(templateContent);
```

目前，多数浏览器已经原生支持了 template：

![img](https://cdn-images-1.medium.com/max/1600/0*3SRCEtv7rMhWpB5s)

如果与自定义 HTML 元素结合，template 将更具威力。我们可以使用 `customElements` 来自定义 HTML 元素：

```js
customElements.define('my-paragraph',
 class extends HTMLElement {
   constructor() {
     super();

     const template = document.getElementById('my-paragraph');
     const templateContent = template.content;
     const shadowRoot = this.attachShadow({mode: 'open'}).appendChild(templateContent.cloneNode(true));
  }
});
```

在元素的构造函数中，我们通过 template 初始化了元素内容，并且内容还被添加到了 shadow DOM，以保证隔离性。现在，我们可以这样定义我们的 `<my-paragraph>` 模板：

```html
<template id="my-paragraph">
  <style>
    p {
      color: white;
      background-color: #666;
      padding: 5px;
    }
  </style>
  <p>Paragraph content. </p>
</template>
```

当中声明的样式会被 scoped。使用这个元素也异常简单：

```html
<my-paragraph></my-paragraph>
```

## Slots

template 的一个不好的地方在于，我们无法在其中声明需要动态创建的内容，此时，我们就需要 **slot**。可以把 slot 理解为占位符：

```html
<template id="my-paragraph">
  <p> 
    <slot name="my-text">Default text</slot> 
  </p>
</template>
```

如果浏览器不支持 slot，那么 slot 部分的内容将回退为 `Default text`。当我们使用自定元素时，就可以插入一个带有 slot 属性的 HTML 元素来动态创建内容，slot 属性标明了该内容对应到哪个占位符：

```html
<my-paragraph>
 <span slot="my-text">Let's have some different text!</span>
</my-paragraph>
```

浏览器渲染的扁平 DOM 树将是：

```html
<my-paragraph>
  #shadow-root
  <p>
    <slot name="my-text">
      <span slot="my-text">Let's have some different text!</span>
    </slot>
  </p>
</my-paragraph>
```

## Scoped CSS

shadow DOM 最优秀的特性即在于 Scoped CSS，这可以：

- shadow DOM 以外的 CSS 选择器无法作用到 shadow DOM 中
- shadow DOM 内部使用的 CSS 选择器也不会影响外部页面

```html
#shadow-root
<style>
  #container {
    background: white;
  }
  #container-items {
    display: inline-flex;
  }
</style>

<div id="container"></div>
<div id="container-items"></div>
```

## :host 伪类

```html
<style>
  :host {
    opacity: 0.4;
  }
  
  :host(:hover) {
    opacity: 1;
  }
  
  :host([disabled]) { /* style when host has disabled attribute. */
    background: grey;
    pointer-events: none;
    opacity: 0.4;
  }
  
  :host(.pink) > #tabs {
    color: pink; /* color internal #tabs node when host has class="pink". */
  }
</style>
```

`:host` 能声明顶层元素的样式，上面这段样式，当 `<extend-button disabled>`时，组件的背景色就将是灰色。

## :host-context(<selector>)

`:host-context(selector)` 则可以声明当前元素所在的上下文，当满足这个上下文时，就能够响应当中声明的样式。其一大应用就是主题：

```html
<body class='lightheme'>
  <custom-container>
  </custom-container>
</body>
```

```css
:host-context(.lightheme) {
  color: black;
  background: white;
}
```

## 在外部声明组件宿主元素样式

在外部，我们可以声明自定义元素的样式：

```css
custom-container {
  color: red;
}
```

它将覆盖组件内部的样式声明：

```css
:host {
  color: red;
}
```

## 使用 CSS 自定义属性创建 style hooks

我们还可以在组件中使用 CSS 自定义属性来创建 style hooks：

```html
<!-- main page -->
<style>
  custom-container {
    margin-bottom: 60px;
     - custom-container-bg: black;
  }
</style>

<custom-container background>…</custom-container>
```

这里的 `custom-container-bg` 相当于被确定内容的 `<slot>` ：

```css
:host([background]) {
  background: var(- custom-container-bg, #CECECE);
  border-radius: 10px;
  padding: 10px;
}
```

这里，组件就将使用黑色作为背景色，因为我们在样式中声明了 `custom-container-bg`。倘若我们没有声明这个值，背景色就默认为 `#CECECE`。

## slotchange 事件

`slotchange` 事件会在 slot 节点变化时被触发：

```js
const slot = this.shadowRoot.querySelector('#some_slot');
slot.addEventListener('slotchange', (e) => {
  console.log('Light DOM change')
})
```

## `assignedNodes()`

调用 `slot.assignedNodes()` 将返回 slot 渲染的元素。

```html
<my-container>
  <span slot="slot1"> container text </span>
</my-container>
```

这里如果我们调用 `assignedNodes`，就将返回 `[<span slot="slot1"> container text </span>]`。

如果我们清空内容：

```html
<my-container></my-container>
```

再调用 `assignedNodes`  ·，就将返回：`[]`。如果我们传递参数 `assignedNodes({flatten: true})`，就将返回 `[<p>Default content</p>]`。

## 事件模型

shodow DOM 中下述事件会进行冒泡：

- Focus Event
- Mouse Event
- Wheel Event
- Input Event
- Keyboard Event
- Composition Event
- Drag Event

而自定义事件则默认不会向外传播。除非你手动声明了 `{bubbles: true}` 以及 `{composed: true}`：

```js
const container = this.shadowRoot.querySelector('#container');
container.dispatchEvent(new Event('containerchanged', {bubbles: true, composed: true}));
```

## 兼容性检测

如果要检测浏览器是否支持 shadow DOM，可以：

```js
const supportShadowDOMV1 = !!HTMLElement.prototype.attachShadow;
```

![img](https://cdn-images-1.medium.com/max/1600/0*k0vSOmvdDkRJzcpW)

