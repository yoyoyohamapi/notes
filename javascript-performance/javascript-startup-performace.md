# JavaScript Start-up Performance

[原文地址](https://medium.com/reloading/javascript-start-up-performance-69200f43b201)

## V8 工作漫画

![](https://cdn-images-1.medium.com/max/2000/1*GuWInZljjvtDpdeT6O0emA.png)

## 观察编译 & 解析性能

1. 使用 DevTools：

   ![](https://cdn-images-1.medium.com/max/2000/0*rWkYJzc6Cp0r3Xkr.)

2. 使用 Chrome Tracing，[参考文档](https://docs.google.com/presentation/d/1Lq2DD28CGa7bxawVH_2OcmyiTiBn74dvC6vn2essroY/edit#slide=id.g1a504e63c9_2_84)

## Script Streaming

浏览器必须先通过解析 HTML 标记来构建 DOM 树，然后才能呈现网页。 在此过程中，每当解析器遇到脚本时，它都必须先停止解析 HTML 并执行该脚本，然后才能继续解析。

![](https://lh6.googleusercontent.com/xRufTH1bAKajbsi3KubVC4KrvGdAM564rc-UzWa6GOJvYHEWF1lUlgUU6uCZb6PDMSfIr8e-diBrB8J3WIGEYM52awXiLtYVdKbMHZXk15r6i--YQmJ2kM7iaUFBP4Hp07Up1VQ)

### 异步加载脚本

可以通过 **`async`** 属性将 script 设置为异步加载：

```js
<script async src="my.js">
```

但是异步脚本未必会按指定的顺序执行，且不应使用 `document.write`。

## 延迟加载脚本

可以通过 **`defer`** 属性 script 设置为延迟加载：

```js
<script defer src="my.js">
```

这样声明的脚本加载时机为：文档加载完毕，`DOMContentLoaded` 事件调用前。且脚本的执行顺序和声明顺序保持一致。

## 我们可以做什么来提升编译 & 解析性能

1. 减少 JavaScript 代码体积
2. 使用 Script Streaming
3. 评估一下所用依赖的解析开销，例如为了追求性能和体积，可以替换 React 为 Preact。
4. 使用 PRPL 模式

## PRPL 模式

PRPL 是一种用于结构化和提供 Progressive Web App (PWA) 的模式，该模式强调应用交付和启动的性能。 它代表：

- **推送（Push）** - 为初始网址路由推送关键资源。
- **渲染（Render）** - 渲染初始路由。
- **预缓存（Pre-cache）** - 预缓存剩余路由。
- **延迟加载（Lazy-load）** - 延迟加载并按需创建剩余路由。



