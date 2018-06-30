# 渲染引擎和优化指南

浏览器的主要组件的构成：

![](https://cdn-images-1.medium.com/max/1600/1*lMBu87MtEsVFqqbfMum-kA.png)

- **UI**：地址栏、向前/返回按钮、书签菜单等等。

- **浏览器引擎（Browser engine）**：处理了 UI 和 渲染引擎的交互。

- **渲染引擎（Rendering engine）**：负责 web 页面的展示。渲染引擎解析了 HTML 和 CSS 内容，并将渲染结果反映到屏幕上。

- **网络（Networking）**：不同的浏览器可能有不同的网络实现，例如不同的 XHR 请求的实现。

- **UI 后端（UI backend）**：负责绘制诸如窗口这样的核心 widget。界面后端暴露了非平台特有的通用接口，底层调用的是操作系统的 UI 方法。

- **JavaScript 引擎（JavaScript engine）**：如 V8 等。

- **数据持久化（Data persistence）**：如 localStorage、indexDB、WebSQL 和 FileSystem。

  ​

## 渲染引擎

- **Gecko** — Firefox
- **WebKit** — Safari
- **Blink** — Chrome, Opera

渲染过程：

![](https://cdn-images-1.medium.com/max/1600/1*9b1uEMcZLWuGPuYcIn7ZXQ.png)

1. 构建 DOM 树
2. 构建渲染树
3. 渲染树布局
4. 绘制渲染树

## 构建 DOM 树

下面这段 html 将会被解析为一棵 DOM 树：

```html
<html>
  <head>
    <meta charset="UTF-8">
    <link rel="stylesheet" type="text/css" href="theme.css">
  </head>
  <body>
    <p> Hello, <span> friend! </span> </p>
    <div> 
      <img src="smiley.gif" alt="Smiley face" height="42" width="42">
    </div>
  </body>
</html>
```

![](https://cdn-images-1.medium.com/max/1600/1*ezFoXqgf91umls9FqO0HsQ.png)

## 构建 CSSOM 树

假定我们的 CSS 内容如下：

```css
body { 
  font-size: 16px;
}

p { 
  font-weight: bold; 
}

span { 
  color: red; 
}

p span { 
  display: none; 
}

img { 
  float: right; 
}
```

引擎也会构建一棵 CSSOM 树：

![](https://cdn-images-1.medium.com/max/1600/1*5YU1su2mdzHEQ5iDisKUyw.png)

这里还会进行一些样式补足，例如 `span` 的字体大小被使用了父元素 `body` 的字体大小进行了补齐。

## 构建渲染树

HTML 中可见的元素及其对应的 CSS 样式数据就将构成一棵渲染树：

![](https://cdn-images-1.medium.com/max/1600/1*WHR_08AD8APDITQ-4CFDgg.png)

在 Webkit 中，渲染树中的每个节点是一个渲染对象。

渲染树的构建过程大致如下：

- 从 DOM 树的根节点开始，遍历每一个**可见的**节点，注入 srcipt 标签，meta 标签等节点是不可见的，CSS 中被设置了 `display: none` 的节点也是不可见的。
- 对于每一个可见的节点，浏览器找到对应的 CSSOM 规则并应用之。
- 浏览器发出可见的节点内容及其样式。

在 Webkit 源码中，每个渲染对象的内容为：

```c++
class RenderObject : public CachedImageClient {
  // Repaint the entire object.  Called when, e.g., the color of a border changes, or when a border
  // style changes.
  
  Node* node() const { ... }
  
  RenderStyle* style;  // the computed style
  const RenderStyle& style() const;
  
  ...
}
```

## 排列

计算渲染对象（renderer）的**尺寸**和**位置**信息的过程被称之为排列（layout） 。

HTML 使用了**基于流**的布局模型，这意味着其能够一次性计算几何信息。坐标系统是关于**根渲染对象（root render）的**，使用 Top 和 Left 坐标。

Layout 是递归完成的，从 root renderer （<hmtl>） 开始，root renderer 的 坐标是 `0,0` ，并且其尺寸包含了 viewport 信息（浏览器窗口能展示网页的区域）。

Layout 过程将指定各个节点出现在屏幕上的位置。

## 绘制

在这一阶段，会遍历渲染树，并且调用渲染对象的 `paint()` 方法，来讲渲染对象的内容展示到屏幕上。类似于 layout 过程，绘制可以是全局式的（global），也可以是增量式的（incremental）：

- **Global**：整个渲染树被绘制
- **Incremental**：部分 renderer 的改变不会影响整个渲染树。renderer 会 invalidate 其在屏幕上的矩形。这会让 OS 认为这个区域需要被重绘并产生一个 `paint` 事件。OS 能够将多个区域合并为一个区域

## 优化

优化渲染性能需要从下面几个方面考虑：

1. **JavaScript**：JavaScript 代码要尽可能不阻塞主线程，并且内存利用足够高效。
2. **样式计算**：这个过程通过选择器，决定了那个 CSS 规则会被应用到待渲染元素上。
3. **排列**：web 的布局模型会造成元素间的相互影响。例如，`<body>` 的宽度会影响到它自己以及它的孩子的布局。我们要牢记，布局是计算密集的任务。
4. **绘制**：这个过程包括绘制文本、颜色、图像、边框、阴影等等。
5. **组合**：页面的渲染需要考虑到元素的层级，和顺序。

### 优化 JavaScript

1. 避免使用 `setTimeout` 或者 `setInterval` 进行可视化的更新。因为它们的回调发生时机是不确定的，可能造成丢帧。
2. 将耗时的 JavaScript 计算交给 Web Worker。
3. 做各个帧间，使用微任务进行 DOM 的变更。我们可以考虑将大的更新任务划分为小的任务，并在  `requestAnimationFrame` , `setTimeout`, `setInterval` 等函数中执行任务。

### 优化 CSS

1. 减少选择器的复杂度，以加快规则搜索的速度。
2. 减少元素样式计算出现的次数。

### 优化 Layout

1. 尽可能减少 layout 的次数。
2. 使用 `flexbox` ，其性能较高。
3. 避免同步的 layout。我们需要牢记的是，当 JavaScript 运行时，所有前一帧的布局数据你都能访问到。如果你直接访问 `box.offsetHeight`，问题不大。但是如果你在访问之前修改了 box 的样式，浏览器首先会应用样式到 box，再运行 layout，以便让你知道最新的 box 尺寸和位置信息。

### 优化绘制

- 任何样式属性的改变都会引起重绘，尽可能少的改变元素样式。
- 如果你调用了一次 layout，你也就会调用一次重绘。
- 通过合理组织动画等方式减少绘制区域。