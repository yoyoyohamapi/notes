# 内存管理以及 4 个常见的内存泄露

什么内容会被放到内存：

1. 程序使用的变量以及一部分数据
2. 程序代码，包括操作系统的代码

## 内存生命期

内存的生命期可以概括为：分配、使用和释放

![](https://cdn-images-1.medium.com/max/1600/1*slxXgq_TO38TgtoKpWa_jQ.png)

## 静态分配

在编译阶段，编译器会根据变量的原始数据类型计算所需内存，并在之后将变量分配到对应大小的**栈空间（stack space）**。之所以称之为栈空间，是因为当函数调用以后，新增的内存被放到了当前内存顶部，当函数结束时，内存又被以**后进先出（LIFO）**的顺序回收。看到下面的代码：

```c
int n; // 4 字节
int x[4]; // 含有 4 个元素的数组，每个元素 4 个字节
double m; // 8 字节
```

编译器在编译这段代码时，能够算出其需要 4 + 4 × 4 + 8 = 28 个字节。之后，编译器便会向操作系统索要 28 个字节的栈空间来存储这些变量。

实际上，当我们写了一个变量 `n`，编译器会将其翻译为类似 “内存地址是 4127963” 这样的信息。

在这个例子中，如果我们尝试访问不存在的 `x[4]`，我们实际上访问到了在内存中紧随其后的 `m`，读写这样不存在的变量将对程序造成破坏：

![](https://cdn-images-1.medium.com/max/1600/1*5aBou4onl1B8xlgwoGTDOg.png)

当函数调用了其他函数，这些函数都会在调用时获得对应的栈空间上的块。它们将自己的局部变量存储在上面，当然，也存储了一个程序计数器用于标识正在执行指令的地址。当函数运行完成，其内存块会被释放。

## 动态分配

光有静态分配是不够的，一些变量占用的空间需要在运行时才知道：

```
int n = readInput(); // 从用户处获得输入
...
// 创建一个长度为 n 的数组
```

在编译阶段，因为尚未获得用户输入，因此就无法确定待创建的数组规模，也就无法为其申请内存空间。

运行时阶段，内存被分配到了**堆空间（heap space）**，静态分配和动态分配的区别如下：

| 静态分配          | 动态分配         |
| ------------- | ------------ |
| 变量大小必须在编译期就知道 | 变量大小在编译期无法确定 |
| 在编译期分配内存      | 在运行期分配内存     |
| 变量被分配到栈       | 变量被分配到堆      |
| 分配顺序为 FILO    | 分配顺序位置       |

## JavaScript 中的内存分配

与 C 等低级语言不通（C 需要 `malloc()`、`free()` 来手动控制内存的分配和释放），JavaScript 为开发者完成了这种 “繁复” 的工作：

```js
var n = 374; // 为数字分配内存
var s = 'sessionstack'; // 为字符串分配内存 
var o = {
  a: 1,
  b: null
}; // 为对象及其属性分配内存
var a = [1, null, 'str'];  // 为数组及其元素分配内存
function f(a) {
  return a + 3;
} // 为函数(实际上是一个可调用对象)分配内存
// 函数表达式也会分配一个对象
someElement.addEventListener('click', function() {
  someElement.style.backgroundColor = 'blue';
}, false);

```

函数调用也会造成对象分配：

```js
var d = new Date(); // 分配了一个 Date 对象
var e = document.createElement('div'); // 分配了一个 DOM 元素
```

对象方法也会分配新的值或者对象：

```js
var s1 = 'sessionstack';
var s2 = s1.substr(0, 3); // s2 是一个新的字符串
// 由于字符串是不可变的，
// JavaScript 可能决定不分配内存，而是直接存储了 [0, 3] 范围的字符串
var a1 = ['str1', 'str2'];
var a2 = ['str3', 'str4'];
var a3 = a1.concat(a2); 
// 连接了 a1 和 a2，返回了一个具有 4 个元素的数组
```

## GC（垃圾回收）

### 内存引用

一个对象如果能够显式或隐式地访问另一个对象，就称该对象引用了另一个对象。例如，一个 JavaScript 对象引用到了它的原型 proptotype，这是**隐式引用**，也引用了其属性，这是**显式引用**。

这里的对象不只指代 JavaScript 中的 “object”，也包含了函数作用域，或者说是**词法作用域（Lexical scope）**。

> 词法作用域定义了嵌套函数中的变量名如何被解析。即便父函数已经 return 了，内部函数还是会引用到父函数的作用域。

### 引用计数法

引用计数法是最简单 GC 算法，当一个变量没有被引用时，它就会被回收。

```js
var o1 = {
  o2: {
    x: 1
  }
};
// 这里创建了两个对象
// 'o2' 被 'o1' 作为属性引用
// 这两个变量都无法被垃圾回收

var o3 = o1; // 变量 'o3' 有一个到 'o1' 指向对象的引用 
                                                       
o1 = 1;      
// 现在，'o1' 所指向的对象只在 'o3' 中存有一个应用

var o4 = o3.o2; 
// 'o4' 有一个到 'o2' 指向对象的引用，
// 'o2' 所指向的对象有两个引用，一个是其属性，一个是 'o4'

o3 = '374'; 
// 现在，一开始 'o1' 指向的对象不再被任何对象引用，因此可以被 GC。
// 但是，'o2' 指向的对象仍然被 'o4' 引用，因此该对象不能被清除。
o4 = null; 
// 现在，一开始 'o2' 指向的对象也不再被任何对象引用，它可以被 GC。
```

#### 循环引用问题

看到下面这段代码，出现了一个循环引用，因此 `o1` 和 `o2` 的引用计数即便在函数运行完成也没有清零，因此二者无法被回收：

```
function f() {
  var o1 = {};
  var o2 = {};
  o1.p = o2; // o1 引用了 o2
  o2.p = o1; // o2 引用了 o1。这造成了一个引用环
}

f();
```

![](https://cdn-images-1.medium.com/max/1600/1*GF3p99CQPZkX3UkgyVKSHw.png)

### 标记清除法

标记清除法的步骤如下：

1. 确定根节点：根节点级标记清除法的起点，在浏览器中，根节点是 `window`，在 Node 中，则是 `global`。
2. 算法遍历根节点及其子孙，标记每个节点为 active。未被标记的节点被认为是垃圾。
3. 最后，垃圾回收器将所有未被标记为 active 的对象回收，释放其内存给 OS。

![](https://cdn-images-1.medium.com/max/1600/1*WVtok3BV0NgU95mpxk9CNg.gif)

该算法的先进性在于，”一个对象有 0 个引用“ 会导致该对象不可达。但是反过来，一个对象不可达，未必它就没有被引用，循环引用就是佐证。因此，该算法并非是垃圾回收部分算法的提升，而是在判定对象可达性上的提升。

现在，循环引用也不会造成问题，因为从根节点出发，二者皆不可达：

![](https://cdn-images-1.medium.com/max/1600/1*FbbOG9mcqWZtNajjDO6SaA.png)

### GC  的不确定性

GC 是不可预测的，这意味着我们无法知道何时会进行 GC。多数 GC 实现会在内存分配阶段进行垃圾回收，如果接下来没有内存分配，GC 就会原地待命。考虑到下面一种场景：

1. 执行了一次大规模的分配。
2. 这是，多个元素被标记为了不可达（例如我们手动将元素指向了 null）。
3. 之后，不再进行内存分配。

在该场景下，大多数 GC 之后都不再进行任何收集工作。即便存在着这些不可达的引用，也不会被回收器回收。

## JavaScript 中的内存泄露

内存泄露指的是我们不再使用的对象仍然驻留在了内存当中，没有被释放。在 JavaScript 中，存在着 4 种典型的内存泄露场景。

### 全局变量

在 JavaScript 中，如果一个变量没有被声明却被引用了，那么该对象会被创建到**全局对象**上，浏览器中就是 `window`。比如下面的代码：

```js
function foo(arg) {
    bar = "some text";
}
```

等同于：

```js
function foo(arg) {
    window.bar = "some text";
}
```

全局变量会耗费额外的内存，而且不能被垃圾回收。如果我们确实需要用全局变量存储信息，一定要保证在不使用它时**将其设置为 `null`**。

### 计时器和回调

```js
var serverData = loadData();
setInterval(function() {
    var renderer = document.getElementById('renderer');
    if(renderer) {
        renderer.innerHTML = JSON.stringify(serverData);
    }
}, 5000);
```

上面这段代码中，由于计时器每个 5s 会调用其回调一次，因此回调引用的 `serverData` 将不能被回收。在使用一些**观察者/订阅者模式的库**或者**接受回调作为参数的函数**时，一定要做好观察或者监听结束时的收尾工作：

```js
var element = document.getElementById('launch-button');
var counter = 0;
function onClick(event) {
   counter++;
   element.innerHtml = 'text ' + counter;
}
element.addEventListener('click', onClick);
// 收尾
element.removeEventListener('click', onClick);
element.parentNode.removeChild(element);
```

### 闭包

```js
var theThing = null;
var replaceThing = function () {
  var originalThing = theThing;
  var unused = function () {
    if (originalThing) // 此处引用了 'originalThing'
      console.log("hi");
  };
  theThing = {
    longStr: new Array(1000000).join('*'),
    someMethod: function () {
      console.log("message");
    }
  };
};
setInterval(replaceThing, 1000);
```

在使用闭包时，我们需要知道，在 JavaScript 中，**一旦一个针对某个闭包的作用域被创建，在同一个父作用域的闭包都将共享该作用域**。

在这个例子中，在 `replaceThingh` 这个回调中，使用 `originalThing` 暂存了 `theThing`；而 `theThing` 下的闭包 `someMethod` 又会共享为了 `unused` 闭包而创建的作用域。 `unused` 作为一个并没有什么卵用的函数，引用了外部变量 `originalThing`，考虑到 `theThing.someMethod` 可以在 `replaceThing` 外部使用，因此，其共享的作用域内的 `originalThing` 就不会被销毁。在这个例子中，每一个 `originalThing` 都会初始化一个大数组，因此，如果我们在浏览器中执行这段代码，并通过性能检测工具检阅，可以看到不断攀升的内存占用。

### DOM 节点的引用

很多开发者喜欢在数组或者对象中保存一份 DOM 节点的引用。这会造成一个 DOM 节点保存了两份引用，其中一份在数组或者对象字典中，一份在 DOM 树上，因此，即便我们删除了 DOM 树上对应节点，对象中还是留有了一份引用，因此节点占用的内存没有被回收：

```js
var elements = {
    button: document.getElementById('button'),
    image: document.getElementById('image')
};
function doStuff() {
    elements.image.src = 'http://example.com/image_name.png';
}
function removeImage() {
    document.body.removeChild(document.getElementById('image'));
    // 此时，image 节点仍在 'elements' 留有引用，因此无法被 GC。
}
```

假定你在代码中保存了一个 `<td>` 节点的引用，而后删除了 DOM 树上的 `<table>` 。你以为除了 `<td>` 以外的 都会被 GC，其实不然，由于 `<td>` 是 `<table>` 上的一个孩子节点，这个节点也保存了它的父亲的引用，因此，`<tbale>` 并不会被 GC，亦即，**一个到 `<td>` 的引用，却导致整个 `<table>` 被放入了内存**。





