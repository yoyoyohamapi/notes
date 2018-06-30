# WebAssembly 简介

WebAssembly 是一个为了 web 而创建的高效的低级字节码格式的内容。其允许你使用非 JavaScript 语言撰写 web 程序，只需要在之后将其编译为 WebAssembly 即可。

其帮助 web 应用更加快速的加载和执行。

## 加载时间

由于 wasm 的内容是类似于汇编的非常紧凑的二进制格式，因此其加载速度会得到保证。

## 执行

V8 的工作流如下：

![](https://cdn-images-1.medium.com/max/1600/0*bN9YVBLw_tT1Xvte.)

JavaScript 代码被解析为了抽象语法树（AST），进而被转化为了未经优化的机器码。

引入了 TurboFan 的 V8 作为优化编译器后，它能监控代码中执行较慢的函数，并将其置入一个 JIT，该 JIT 通过压榨 CPU 性能来优化这些函数：

![](https://cdn-images-1.medium.com/max/1600/0*wzuQ9LYv7CAUICOC.)

但这种通过 CPU 进行优化的手段也会带来更高的电量损耗，这对移动设备影响很大。

wasm 则在**编译期**就完成了优化，也省略了将源码解析为 AST 的过程。一份优化后的代码将直接送入 JIT 转化为机器码：

![](https://cdn-images-1.medium.com/max/1600/0*GDU4GguTzk8cSAYk.)

## 内存模型

![](https://cdn-images-1.medium.com/max/1600/0*QphcOVaiVC2YL7Jd.)

WebAssembly 的执行栈与 WebAssembly 程序本身是分离的，所以你无法改变其内部的变量。函数也是使用的整数偏移，而不是指针。函数指向了一个间接的函数表。而后，可以直接计算出模块中函数的位置，并跳到该位置。这让你能够一次加载多个 wasm 模块，这些模块中的函数各自有其偏移量，不会冲突。

## GC

WebAssembly 使用撰写语言本身提供的 GC 能力。例如 C 可以用 malloc，C ++ 用 smart pointer，Rust 则有 ownership。

## 平台 API 的访问

WebAssembly 不能访问平台提供的 API，例如不能访问浏览器提供的 DOM、WebGL 等 API。

## SourceMap

WebAssembly 目前还不支持 SourceMap。

## 多线程

WebAssembly 尚不支持多线程。

## 适用性

WebAssembly 适用于一些计算密集型的任务，例如数字图像处理等等。

## h