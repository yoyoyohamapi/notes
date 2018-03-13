# Rust Ownership

## 栈（Stack）

变量被逐个放入（push）栈中，然后再以相反顺序取出（pop），即先进后出）。

栈的速度是很快的，这是因为如下两个原因：

1. 数据的存取是严格遵循规定的，我们不需要搜索数据存放的位置，不断放入，不断从顶部取出即可。
2. 栈里面的每一个数据都是大小固定的。

## 堆（Heap）

如果数据的大小未知，或者之后会改变，那么变量就会被存在堆上。相对于栈，堆的组织较为松散。当我们向堆请求一定的空间存放数据时，操作系统会为我们找到一块足够大的区域，并将其标记为已使用，并返回给我们一个**指针**，该指针指向了这块堆的位置。该过程被称为**堆上分配**，也可以简称为分配（allocate），而栈的分配则倾向于使用放入（push）。由于指针是一个已知的、大小固定的数据结构，因此指正可以被存储到栈上，当我们要获得堆上的数据时，先要通过指针带路。

在堆上访问数据比较慢，这是因为我们需要循着指针才能到对应的堆区域。举个例子，餐馆里面的服务员会收取每一张桌子上的订单。最快的收集方式是，每次收集完整个桌子的，再去收集下一张桌子。如果一张桌子收一点儿，效率就很慢。处理器亦是如此，栈上的数据紧凑，处理器收集数据就快，堆上的数据松散，处理器收集数据的速度就慢。另外，在堆上分配一个大空间也需要花费时间。

Rust 引入的所有权机制能帮助解决下面这些问题：

- 跟踪代码中的哪些部分正在只用堆上的数据
- 最小化堆上的冗余数据
- 清除堆上没有使用的数据

## 所有权规则

Rust 中的所有权规则如下：

- Rust 中的每一个值都对应一个变量，该变量被叫做这个值的所有者（owner）
- 同一时刻，值的所有者只能有一个
- 当所有者离开了作用域，值就要被释放

## 变量作用域

作用域限定了某个元素的有效范围。

```rust
{                      // s is not valid here, it’s not yet declared
    let s = "hello";   // s is valid from this point forward

    // do stuff with s
}  
```

这里注意两点：

1. 当 `s` 进入作用域，它就变得有效。
2. `s` 将一直保存，直到其离开作用域。

## 内存分配

### 通过字符串字面量创建字符串：

```rust
let s = "hello";
```

字符串字面量高效快速的原因是，其不可变性使得它能在编译器就完成内存分配。

### 通过 `String` 类型创建字符串：

```rust
let mut s = String::from("hello");

s.push_str(", world!"); // push_str() appends a literal to a String

println!("{}", s); // This will print `hello, world!`
```

在这个例子中，为了支持字符串的内容可变，就需要：

1. 必选在运行时向操作系统请求内存。
2. 当工作完成时，我们需要返回内存给操作系统。



在使用了 GC 的语言中，程序员不需要关心内存的清除，GC 会跟踪变量并且清除不再使用的内存。而在没有 GC 的语言中，开发者需要手动清理内存。Rust 不同于两者，其规定了：

> 如果变量离开了它的作用域，它就需要被清空。

例如下面这段代码，Rust 会在 `s` 离开作用域时，清空其所占的内存：

```js
{
    let s = String::from("hello");
}
```

这里实际上调用了 `drop` 函数进行内存清理，`String` 也有对应的 `drop` 实现。

## 变量和数据的交互方式：移动（Move）

看到下面的代码：

```rust
let x = 5;
let y = x;
```

由其他语言的知识，我们不难推测：

1. 将值 `5` 绑定到变量 `x` 上；
2. 创建一份 `x` 的拷贝，并将绑定该拷贝到 `y` 上。

实际上也确实如此，由于 `5` 是已知且大小固定的，所以两个 `5` 都被 push 进了栈中。

但再看到下面的代码：

```rust
let s1 = String::from("hello");
let s2 = s1;
```

我们不能想当然的认为其工作过程和之前的代码一致。

一个 `String` 类型的变量由三部分信息分组成：

- 指针：指向堆上的字符串内容
- 长度：当前字符串用的长度，单位为字节
- 容量：当前字符串占用的内存大小，单位为字节

这三部分因为已知且大小固定，因此被保存在了栈上。指针指向的内容则保存在了堆上。

![](https://doc.rust-lang.org/book/second-edition/img/trpl04-01.svg)

当我们将 `s2` 赋值给 `s1`，我们会拷贝栈上保存的信息，而共享堆上的内容：

![](https://doc.rust-lang.org/book/second-edition/img/trpl04-02.svg)



Rust 不会也复制堆上的内容，就像下图一样，因为假如数据很大，进行堆上的复制会带来高昂的运行时开销：

![](https://doc.rust-lang.org/book/second-edition/img/trpl04-03.svg)

这里会存在一个问题，当 `s2` 和 `s1` 都离开了作用域，那么二者都会被清空以释放内存。二次清除操作会导致内存安全的 bug。

为了确保内存安全，Rust 不会复制已经分配的内存。Rust 会认为，有了 `s2`，则 `s1` 不再需要，也就不再有效了。当 `s1` 离开了作用域，Rust 不需要做任何清除操作。下面这段代码展示了如果我们在 `s2` 创建之后继续使用 `s1` 会发生什么：

```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{}, world!", s1);
```

程序将报错，Rust 不允许你再继续使用 `s1` 了：

```bash
error[E0382]: use of moved value: `s1`
 --> src/main.rs:5:28
  |
3 |     let s2 = s1;
  |         -- value moved here
4 |
5 |     println!("{}, world!", s1);
  |                            ^^ value used here after move
  |
  = note: move occurs because `s1` has type `std::string::String`, which does
  not implement the `Copy` trait
```

只拷贝元信息（指针、长度、容量），而不拷贝数据，这样的拷贝可以叫做是浅拷贝。Rust 在浅拷贝的基础上海多了一步：使 `s1` 无效化，因此，Rust 不将这个过程叫做浅拷贝，而是命名为**移动**。

![](https://doc.rust-lang.org/book/second-edition/img/trpl04-04.svg)

此时，Rust 只需要在 `s2` 离开作用域时清空其内存。

## 变量和数据的交互方式：克隆（clone）

如果真的想要深度拷贝堆上的数据，Rust 也为 `String` 类型提供了 `clone()` 方法：

```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("s1 = {}, s2 = {}", s1, s2);
```

## 只在栈上的数据：拷贝（Copy）

回顾到之前的代码：

```rust
let x = 5;
let y = x;

println!("x = {}, y = {}", x, y);
```

在这段代码中，由于 `5` 是已知且大小固定的，因此数据是在栈上存储的， `x` 也就不需要移动到 `y`，而是直接进行栈数据的拷贝。

Rust 通过类型上的 `Copy` trait 进行栈上的数据拷贝。如果一个类型含有 `Copy` trait，那么旧的变量在赋值后仍可以使用。如果某个类型或者该类型的某个部分已经实现了 `Drop` trait，Rust 就不允许再声明 `Copy` trait。如果类型需要在离开作用域时进行一些处理，再为其声明 `Copy` 特性就会导致编译错误。

## 所有权与函数

传递某个值给函数类似于将某个值赋给变量，要么进行移动，要么进行拷贝。

```rust
fn main() {
    let s = String::from("hello");  // s 进入作用域。

    takes_ownership(s);             // s 的值被移入了函数，因此其不再有效。

    let x = 5;                      // x 进入了作用域。

    makes_copy(x);                  // x 的值被拷贝到了函数，因此后续能够继续使用 x。

} 
// 此处，x 先离开作用域，s 后离开作用域，由于 s 已经被移动，因此不会有额外操作。

fn takes_ownership(some_string: String) { // some_string 进入作用域。
    println!("{}", some_string);
} 
// 此处，some_string 离开作用域并且 `drop` 被调用。其占用的内存被释放。

fn makes_copy(some_integer: i32) { // some_integer 进入作用域。
    println!("{}", some_integer);
} 
// 此处，some_integer 离开作用域。由于其类型已经具有了 Copy trait，就不再有 Drop trait，因此什么都不会发生。
```

## 返回值与所有权

返回值也能转移所有权。

```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership 将其返回值的所有权移到了 `s1`

    let s2 = String::from("hello");     // s2 进入作用域

    let s3 = takes_and_gives_back(s2);  // s2 的所有权被移动到了 takes_and_gives_back, 
                                        // 该函数又将其返回值的所有权移动到了 `s3`
} 
// 此处，s3 离开了作用域并被 drop 掉。s2 离开了作用域，但由于其移动过，因此无需 drop。
// s1 离开作用域并被 drop。

fn gives_ownership() -> String {             // gives_ownership 将会把它的返回值所有权
                                             // 移交给调用它的函数

    let some_string = String::from("hello"); // some_string 进入作用域。

    some_string                              // some_string 的所有权被转交给了该函数的调用者
}

// takes_and_gives_back 接受一个字符串，并且返还其所有权
fn takes_and_gives_back(a_string: String) -> String { // a_string 进入作用域

    a_string  // a_string 的所有权被转交给了该函数的调用者
}
```

## 参考资料

- [4.1 What is ownership?](https://doc.rust-lang.org/book/second-edition/ch04-01-what-is-ownership.html#ownership-and-functions)