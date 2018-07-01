# AST 与 5 个优化解析时间的 tip

几乎所有的编程语言都会先将开发者的**源代码**编译成一个特殊的数据结构：**抽象语法树（Abstract Syntx Tree，AST）**。AST 扮演的角色有：

- 描述源码的数据结构
- 源码的语义化描述，方便编译器校验程序的正确性，以及做出一定优化
- 静态代码分析（Static code analyzing），用于理解代码结构，例如分析并减少冗余代码
- 代码转换，例如将 Python 代码转化为 JavaScript 代码

## JavaScript 解析

下面这段代码：

```js
function foo(x) {
  if (x > 10) {
    var a = 2;
    return a * x;
  }
  
  return x + 10;
}
```

就将生成如下的 AST，这个 AST 是简化过的，实际上复杂得多：

![](https://cdn-images-1.medium.com/max/1600/0*mSOIiWpkctkD0Gfg.)

如果想要查看真正的 AST，那么可以使用 [AST Explorer](https://astexplorer.net/)

下面这张图中，我们可以看到，JavaScript parsing 时间占到了总时间的 15% 到 20%。

![](https://cdn-images-1.medium.com/max/1600/0*eEArxn147Ev8xf5n.)

在移动端的话，受限于机器性能，这个时间还要更长。

## 浏览器引擎的解析优化

V8 会进行 script streaming 以及 code caching。script streaming 意味着**异步（async）**或者**延迟（deferred）**加载的脚本会在下载开始后，就在不同的线程进行加载。也即是脚本加载完成后，解析工作也进行完成了，这会提高约 10% 的页面加载性能。

JavaScript 代码通常会在每个页面被访问时被编译成二进制代码。但是，当我们跳转到其他页面时，这些代码又会被丢弃。这是因为这些代码与编译时的状态和上下文都相关。Chrome 42  引入了二进制代码缓存。这是一个能在本地存储编译后代码的技术，借助于这个技术，将省去页面加载时的下载、解析和编译时间。这让 Chrome 能够节约 40% 的解析和编译时间。另外，这也帮助移动设备节约了 CPU 时间，也就优化了耗电。

在 Opera 中，**Carakan** 引擎能够重用另一个程序中最近用到的编译过的代码。甚至不要求这段代码需要是同一个页面或者同一个域下。这个技术能够完全跳过编译周期。这个缓存技术基于的哲学是用户路径，如果用户在一个应用里保持了相同的轨迹，那么相同的 JavaScript 代码将会被加载。然而，Carakan 引擎早已被 V8 引擎所替代。

FireFox 所使用的 SpiderMonkey 引擎不会进行缓存，而是监控脚本执行频率，这样就能优化哪些经常被执行的脚本。

而像 Safari 的核心开发者，Maciej Stachowiak 则称 Safari 不会缓存任何编译后的二进制代码，因为底层还需要优化，目前的性能提升有限。

## 提早解析与延后解析

现代 JavaScript parser 使用启发式（heuristics）的方式来决定一段代码需要被立即执行还是延后执行。基于此，parser 会决定是要提早（eager）还是延后（lazy）解析。

提前解析立即遍历需要被解析的函数，并完成：

- 构建 AST
- 构建作用域层级（scope hierarchy）
- 发现所有的语法错误

延后解析只在函数还不需要被编译时使用。其不会构建 AST 和找寻语法错误，而是只会构建 scope hierarchy。因此，它比提前解析节约了一般的时间。

两者并不是新的概念，即便是 IE 9，也有了这样的优化手段。

看个例子：

```js
function foo() {
  function bar(x) {
    return x + 10;
  }
  
  function baz(x, y) {
    return x + y;
  }
  
  console.log(baz(100, 200));
}
```

我们逐行分析代码：

函数声明 `bar`  接收一个参数 `x`，返回 `x + 10 `的结果。

函数声明 `baz` 则接收两个参数 `x` 和 `y`。返回 `x + y` 的结果。

进行一次函数调用，`baz(100, 200)`

进行一次函数调用，其参数为 `baz(100, 200)` 的返回值。

最终，生成的 AST 大概会是：

![](https://cdn-images-1.medium.com/max/1600/0*60xiqW7kPsQg5ssn.)

在这段代码中，我们可以看到，函数 `bar` 并没有使用过，这就像我们代码里含有的许多无用代码一样。因此，parser 会对此进行优化，parser 只会进行必要的 parse，并且会在函数执行前进行 parse。lazy parse 仍然会解析函数体，但不会进行处理，研究不会进行堆内存的分配。这会带来明显的性能提升：

![](https://cdn-images-1.medium.com/max/1600/0*IN688nPbgu8zYETe.)

现在，`bar` 在 AST 中，仅仅具有一个声明。

而在下面这个定义模块的代码，parser 就知道要尽早解析：

```js
var myModule = (function() {
  // The whole logic of my module
  // Return the module object
})();
```

为什么不都进行延迟解析呢？这样就不会占用太多的初始化时间？这是因为如果我们都是按需解析，那么在代码执行前，就需要耗费解析时间，亦即执行被延迟了。

通过为函数包裹上一个**括号**，来提醒 parse 进行提前解析：

```js
var foo = (function foo(x) {
  return x * 10;
});
```

但是，如果在代码编写的时候我们人为的来做这些事儿，就会牺牲掉代码的可读性。因此，我们可以借助像 Optimize.js 这样的工具来为我们进行函数包裹。

## 优化代码初始化时间的 Tips

- 检查项目依赖，去除无效依赖
- code splitting
- 按需加载
- 使用开发者工具检查系统的瓶颈所在
- 使用 Optimize.js 来优化 parse，决定哪些代码应该提前解析和延后解析

