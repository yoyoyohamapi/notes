# Advanced Functions & Closures

## 函数指针

通过函数指针，我们能够将函数作为参数，进行传递：

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}

fn main() {
    let answer = do_twice(add_one, 5);

    println!("The answer is: {}", answer);
}
```

不同于闭包，`fn` 是 type 而不是 trait，所以我们可以直接将 `fn` 声明为参数，而不是声明一个实现了 `Fn` trait 的泛型。

函数指针实现了三个闭包 trait（`Fn`、`FnMut` 和 `FnOnce`）。什么时候我们只能使用函数指针呢，就是我们需要和外部那些没有闭包的函数进行交互的时候，例如 C 语言接受函数作为参数，但是 C 语言没有闭包。

下面的例子中，我们既可以使用 inline 闭包，也可以使用具名函数：

```rust
let list_of_numbers = vec![1, 2, 3];
let list_of_strings: Vec<String> = list_of_numbers
    .iter()
    .map(|i| i.to_string())
    .collect();
```

```rust
let list_of_numbers = vec![1, 2, 3];
let list_of_strings: Vec<String> = list_of_numbers
    .iter()
    .map(ToString::to_string)
    .collect();
```

## 返回闭包

由于闭包是 trait，因此在函数中，我们不能直接返回闭包。在大多数我们需要返回 trait 的场景中，我们可以返回一个实现了该 trait 的具体类型。但对于闭包，我们做不到，因为闭包没有可以返回的具体类型：

```rust
fn returns_closure() -> Fn(i32) -> i32 {
    |x| x + 1
}
```

这段代码将编译错误：

```
error[E0277]: the trait bound `std::ops::Fn(i32) -> i32 + 'static:
std::marker::Sized` is not satisfied
 -->
  |
1 | fn returns_closure() -> Fn(i32) -> i32 {
  |                         ^^^^^^^^^^^^^^ `std::ops::Fn(i32) -> i32 + 'static`
  does not have a constant size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for
  `std::ops::Fn(i32) -> i32 + 'static`
  = note: the return type of a function must have a statically known size
```

Rust 不知道要为闭包分配多少空间。解决方式是我们可以使用一个 trait object，`Box` 能让内存分配推迟到运行时： 

```rust
fn returns_closure() -> Box<Fn(i32) -> i32> {
	Box::new(|x| x + 1)
}
```

