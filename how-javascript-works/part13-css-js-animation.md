# CSS 和 JS 动画，及性能优化

## CSS transition

使用 CSS 声明 transition，而使用 JavaScript 来适时地启动 transition。

**.css**：

```css
.box {
  -webkit-transform: translate(0, 0);
  -webkit-transition: -webkit-transform 1000ms;

  transform: translate(0, 0);
  transition: transform 1000ms;
}

.box.move {
  -webkit-transform: translate(50px, 50px);
  transform: translate(50px, 50px);
}
```

**.html**：

```html
<div class="box">
  Sample content.
</div>
```

**.js**:

```js
var boxElements = document.getElementsByClassName('box'),
    boxElementsLength = boxElements.length,
    i;

for (i = 0; i < boxElementsLength; i++) {
  boxElements[i].classList.add('move');
}
```

还可以监听 **`transitioned`** 事件，在 transition 完成后做一些处理：

```js
var boxElement = document.querySelector('.box'); // Get the first element which has the box class.
boxElement.addEventListener('transitionend', onTransitionEnd, false);

function onTransitionEnd() {
  // Handle the transition finishing.
}
```

## CSS animation

keyframe 指出了某个阶段 CSS 样式如何，各阶段之间的间隔由浏览器负责填充。

```css
.box {
  /* Choose the animation */
  animation-name: movingBox;

  /* The animation’s duration */
  animation-duration: 2300ms;

  /* The number of times we want
      the animation to run */
  animation-iteration-count: infinite;

  /* Causes the animation to reverse
      on every odd iteration */
  animation-direction: alternate;
}

@keyframes movingBox {
  0% {
    transform: translate(0, 0);
    opacity: 0.4;
  }

  25% {
    opacity: 0.9;
  }

  50% {
    transform: translate(150px, 200px);
    opacity: 0.2;
  }

  100% {
    transform: translate(40px, 30px);
    opacity: 0.8;
  }
}
```

## JavaScript animation

使用 JavaScript 动画，将能够更细粒度的控制元素样式。比如减缓动画速度、暂停动画等等：

```js
var boxElement = document.querySelector('.box');
var animation = boxElement.animate([
  {transform: 'translate(0)'},
  {transform: 'translate(150px, 200px)'}
], 500);
animation.addEventListener('finish', function() {
  boxElement.style.transform = 'translate(150px, 200px)';
});
```

## Easing

现实世界中，很少有物体保持线性的速度从一个点移动到另一个点。他们总会在运动过程中加速或者减速：

- **ease in**：慢速开始，之后逐渐加速。
- **ease out**：快速开始，之后逐渐减速。

二者也可以混合，例如 **ease in out** 就表示：慢 —> 快 —> 慢。

CSS 支持声明 transition 的运动过程为：

- `linear`
- `ease-in`
- `ease-out`
- `ease-in-out`
- `cubic-bezier`

### `linear`

![](https://cdn-images-1.medium.com/max/1600/1*M5htfOGgza04ISv_l-69zg.png)

```css
transition: transform 500ms linear;
```

### `ease-out`

![](https://cdn-images-1.medium.com/max/1600/1*VDWQl67cmbyAFC5xL9Og4g.png)

```css
transition: transform 500ms ease-out;
```

### `ease-in`

![](https://cdn-images-1.medium.com/max/1600/1*rWh8YlBn8SypiMduLiYDhA.png)

```css
transition: transform 500ms ease-in;
```

### `ease-in-out`

![](https://cdn-images-1.medium.com/max/1600/1*tGXhNroe8KxGN7r4UTVSHw.png)

```css
transition: transform 500ms ease-in-out;
```

### 贝塞尔曲线（Bézier curves）

贝塞尔曲线由四个部分构成：

- 起点
- 终点
- 两个控制端点

调节控制点，将产生不同的曲线：

![](https://cdn-images-1.medium.com/max/1600/1*2v7G1ZJ1C-y_mWHOYQfQKQ.png)

![](https://cdn-images-1.medium.com/max/1600/1*P5nzyldL4rg36dZmt2RViQ.png)

```css
transition: transform 500ms cubic-bezier(0.465, 0.183, 0.153, 0.946);
```

## 性能优化

### `will-change`

在 CSS 中使用 `will-change` 来告诉浏览器我们试图去改变元素的属性。这样浏览器提前为样式变更做好准备，但是滥用 `will-change` 可能会恶化性能：

```css
.box {
    will-change: transform, opacity;
}
```

### JavaScript 和 CSS 的取舍

- CSS 动画是浏览器通过 “compositor thread” 进行支持的，并且与 “main thread” 是分离的，因此，即便浏览器在 “main thread” 上进行耗时任务，动画仍然不会被打断。
- `transform` 和  `opacity` 的变更在多数时候都由 “compositor thread” 负责。
- 任何动画如果触发了重绘（paint）、重排（layout），都将调动 “main thread” 工作。因此，动画在对元素的宽高、颜色、位置、背景调整时，需要额外注意。