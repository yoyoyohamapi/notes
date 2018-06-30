# 引擎、运行时以及调用堆栈

## JavaScript 引擎构成

![](https://cdn-images-1.medium.com/max/800/1*OnH_DlbNAPvB9KLxUCyMsA.png)

- **内存堆（Memory Heap）**：分配内存的地方
- **调用堆栈（Call Stack）**：存放代码执行的堆栈帧

## 运行时

除了 JavaScript 引擎，不同的平台还会提供一些其他特性，如浏览器为我们提供了 WEB API 来操纵 DOM、AJAX、setTimeout 等。另外，还有**事件循环（Event Loop）**、**回调队列（Callback Queue）**等。

![](https://cdn-images-1.medium.com/max/1600/1*4lHHyfEhVB0LnQ3HlhSs8g.png)

## 调用堆栈

看到下面的代码片段：

```js
function multiply(x, y) {
    return x * y;
}
function printSquare(x) {
    var s = multiply(x, x);
    console.log(s);
}
printSquare(5);
```

 调用栈的**帧**进出过程如下：

![](https://cdn-images-1.medium.com/max/1600/1*Yp1KOt_UJ47HChmS9y7KXw.png)

下面这段代码将为我们展示经常在浏览器控制台看到的错误信息堆栈：

```js
function foo() {
    throw new Error('SessionStack will help you resolve crashes :)');
}
function bar() {
    foo();
}
function start() {
    bar();
}
start();
```

![](https://cdn-images-1.medium.com/max/1600/1*T-W_ihvl-9rG4dn18kP3Qw.png)

而下面这段代码则会造成栈溢出：

```js
function foo() {
    foo();
}
foo();
```

![](https://cdn-images-1.medium.com/max/1600/1*AycFMDy9tlDmNoc5LXd9-g.png)

![](https://cdn-images-1.medium.com/max/1600/1*e0nEd59RPKz9coyY8FX-uw.png)