# 从 V8 中了解 5 个代码优化手段

为了提升性能，通过实现了 **JIT（Just-In-Time）编译器**，V8 将 JavaScript 源码在执行阶段转换为机器码，而不是使用解释器。

5.9 版本前的 V8 使用了两个编译器：

* full-codegen：这个编译器非常简单、快速，能产生速度相对慢一些的机器码。
* Crankshaft：一个更加复杂的 JIT 编译器，能够产生高度优化的代码。

V8 内部也有若干线程：

* 主线程：编译 JavaScript 代码，并且执行
* 当主线程执行时，还有另外的一个线程负责编译。
* 还有一个 Profiler 线程（分析器线程）能告诉运行时那个方法很耗时，从而交给 Crankshaft 去优化。
* 另一些线程则负责**垃圾回收（Garbage Collector）**

首次执行 JavaScript 代码时，V8 使用了 **full-codegen** 将 JavaScript 代码编译为机器码，此时不做任何转换。这样能够非常快速的开始执行机器码。当代码运行过几次后，**profiler 线程**就收集到了足够的信息，知道哪些方法应当被优化。

接下来，**Crankshaft** 优化在另一线程中开始。它将 JavaScript 的 AST 转换为一个高级的，叫做 **Hydrogen** 的**静态单一指派**（static single assignment，SSA 要求每个变量只被赋值一次，并且在使用前就被定义），然后尝试去优化 Hydrogen 图。绝大多数的优化都是在这一层面完成的。

 ## V8 的优化手段

### inline

V8 使用的第一个优化手段就是将函数的调用点替代为函数体内的内容：

![](https://cdn-images-1.medium.com/max/1600/0*RRgTDdRfLGEhuR7U.png)

### hidden class

JavaScript 是一个基于原型（prototype）的语言，因此没有类，且对象都是通过克隆手段创建的。另外，作为一门动态语言，JavaScript 中，对象实例化后，其属性可以随意增删。

大多数 JavaScript 解释器使用了类似字典的数据结构来在内存中存储对象属性的位置，该结构会基于 hash 函数。这使得 JavaScript 中的属性检索开销甚大。因此 V8 使用了 **hidden class**。hidden class 类似于 Java 等静态类型语言中的固定对象结构（如 Java 中的类），区别是 hidden class 是在运行时创建的。看到下面的一个例子：

```js
function Point(x, y) {
    this.x = x;
    this.y = y;
}
var p1 = new Point(1, 2);
```

当执行到 `new Point(1, 2)` 时，V8 会创建一个叫做 “C0” 的 hidden class：

![](https://cdn-images-1.medium.com/max/1600/1*pVnIrMZiB9iAz5sW28AixA.png)

由于还没有对这个 Point 对象定义任何属性，因此 “C0” 是空的，一旦 “Point” 函数中的 `this.x = x` 执行了，V8 就会基于 “C0” 创建另一个 hidden class —— “C1”。“C1” 中描述了属性 x 在内存中的位置，该位置不是一个**绝对地址**，而是相对于对象指针的相对位置。在本例中，属性 `x` 被存放在了 0 偏移位置，这意味着，若果在内存中看到一个存储 Point 对象的 buffer，那么这个 buffer 的第一个偏移量就存储了属性 `x`。

V8 也会通过 “类转移（class transition）” 更新 “C0” 为 “C1”，类转移认为，如果一个属性 `x` 被添加到了一个 Point 对象，则 hidden class 应当从 “C0” 转换为 “C1”。现在，Point 对象对应的 hidden class 即为 “C1”：

![](https://cdn-images-1.medium.com/max/1600/1*QsVUE3snZD9abYXccg6Sgw.png)



>  hidden class 的迁移是十分重要的，因为这让以同样方式创建的对象能够共享 hidden class。如果两个对象共享了一个 hidden class，并且添加了相同的属性，类迁移将确保这两个对象都接收到相同的 hidden class，也就就收到了所有这个新的 hidden class 带来的优化代码。

当执行到 `this.y = y` 时，一样有新的 hidden class 被创建，一样会发生类迁移：

![](https://cdn-images-1.medium.com/max/1600/1*spJ8v7GWivxZZzTAzqVPtA.png)

要注意的是，类迁移过程取决于属性添加到对象的属性，下面这段代码中，`p1`、`p2` 由于属性添加顺序的不同，造成了不同的类迁移路径，因此二者最终不会共享相同的 hidden class。这也启示了我们，动态添加属性时，**尽量保持顺序一致**，这样能最大程度重用 hidden class：

```js
function Point(x, y) {
    this.x = x;
    this.y = y;
}
var p1 = new Point(1, 2);
p1.a = 5;
p1.b = 6;
var p2 = new Point(3, 4);
p2.b = 7;
p2.a = 8;
```

### inline 缓存（优化 hidden class 的检索）

考虑到重复地调用某个方法总是发生在同类型的对象，因此，V8 还是用了 inline 缓存技术来提升性能。V8 维护了一个对象类型缓存，这些对象在最近的函数调用中被作为参数传入，V8 认为，同类型的对象会再次被作为参数传入该方法。如果 V8 能够对传入方法中的对象类型作出正确假设，它就能使用缓存中的保存的，对该类型对象的 hidden class 的上一次检索信息来获得对象属性，而避开寻常的对象属性访问过程。

当某个方法调用在一个具体对象上时，V8 通过检索该对象对应的 hidden class 来确定某个属性的偏移量。在对同一 hidden class 的两次成功的方法调用之后，V8 将不再检索 hidden class，仅只将偏移量添加到对象指针自身。那么，之后对于该方法的调用，V8 会认为 hidden class 没有发生改变，通过访问缓存，直接获得上一次属性检索时保存的属性偏移量，从而极大地提升执行性能。

Inline 缓存也再次强调了同类型对象共享相同 hidden class 的重要性。如果你创建了两个不同 hidden class、确是相同类型的对象，V8 将无法利用到 inline 缓存，因为它们各自的 hidden class 对于同一属性，不具有相同的偏移量：

![](https://cdn-images-1.medium.com/max/1600/1*iHfI6MQ-YKQvWvo51J-P0w.png)

### 编译到机器码

一旦 Hydrogen 图被优化了，Crankshaft 就将其转化为被称作是 Lithium 的低级描述。寄存器分配也发生在这一层级。

最终，Lithium 被编译为了机器码。然后 OSR（on-stack replacement：堆栈上替换） 发生了。V8 会将所有我们具有的上下文（栈、寄存器等）在执行期间，转换为对应的优化版本。这种转换也是可逆的，为了在引擎作出错误假设时，能恢复到优化前的版本。

### 垃圾回收

V8 使用了传统的垃圾回收手段 —— 标记清除法。在标记阶段，会停止 JavaScript 的运行。为了提高 GC 性能，V8 优化了标记过程：它不再遍历整个堆，标记每一个可能的对象，而是遍历堆的一部分，然后继续下一次执行。下一次 GC 再继续从上次遍历到的位置开始。这样，每次代码执行时，都会容留小的暂停来做 GC。另外，还会通过多线程来完成清除阶段的工作。

### Ignition 和 TurboFan

V8 的 5.9 版本带来了新的解释器 —— Ignition，以及新的优化编译器 —— TurboFan，从而淘汰了原来的 full-codegen 和 Crankshaft。借此，V8 不仅性能更好，而且更加简单和易于维护：

![](https://cdn-images-1.medium.com/max/1600/0*pohqKvj9psTPRlOv.png)

## 优化我们自己的代码



1. **对象属性顺序**：**按照相同的顺序实例化我们的对象属性**，这有利于共享 hidden class。
2. **动态属性**：由于动态添加对象属性将会引起 hidden class 变换，并减慢为上一个 hidden class 优化过的方法。**尽量在构造方法中分配对象属性**。
3. **方法**：由于 inline 缓存的关系，**重复执行同一个方法**会比执行不同方法一次要快。
4. **数组**：避免稀疏数组（数组的 key 不是自增的），因为稀疏数组实际上是一个 hash 表，因此就避不开代价高昂的哈希查找。同样要**避免预先分配一个大数组**，而应该按需扩展。最后，**不要在数组中进行元素删除**，这会让数组的 key 变得稀疏。
5. **标记值**：V8 使用 32 位来描述对象和数字。其中使用了 1 位标志位来判断它是对象（flag = 1）还是一个 31 位的 SMI （Small Integer）数字（flag = 2）。如果一个数超过 31 位，V8 就会将其转换为 double 值，并且创建一个新的对象保存它。因此，**尽量使用 31 位的带符号数字**，避免装箱操作的开销。