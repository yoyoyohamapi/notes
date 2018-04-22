# Advanced Type

## 使用 Newtype pattern 完成类型安全和抽象

new type 的作用可以有：

1. **类型安全**：强制了 value 类型，因而不会产生混淆：例如我们通过 `Millimeters` 和 `Meters` 包裹了 `u32` value。如果我们在调用一个函数参数是 `Millimeters` 类型的函数时，传入了 `Meters` 或者 plain `u32`，Rust 不会通过编译。
2. **类型抽象**：例如，我们提供了一个 `People` 类包裹了 `HashMap<i32, String>`，其存储了用户 ID 和姓名。和 `People` 类交互的代码只能与其暴露的 public API 交互。例如，有一个方法能够添加一个 name 字符串到 `People` 集合中，这个代码不需要知道我们会在内部为这个 name 分配一个 id。

## 类型别名

类型别名能为现有类型起一个别名：

```rust
type Kilometers = i32;
```

这意味着，`Kilometers` 是 `i32` 的**同义词**，要注意，它不是一个 new type，Rust 会把 `Kilometers` 当做是 `i32`：

```rust
type Kilometers = i32;

let x: i32 = 5;
let y: Kilometers = 5;

println!("x + y = {}", x + y);
```

类型别名的主要作用是减少代码重复，例如，我们有如下冗长的一个类型：

```rust
Box<Fn() + Send + 'static>
```

使用的时候会是：

```rust
let f: Box<Fn() + Send + 'static> = Box::new(|| println!("hi"));

fn takes_long_type(f: Box<Fn() + Send + 'static>) {
    // --snip--
}

fn returns_long_type() -> Box<Fn() + Send + 'static> {
    // --snip--
}
```

借助于类型别名，我们可以大幅精简代码：

```rust
type Thunk = Box<Fn() + Send + 'static>;

let f: Thunk = Box::new(|| println!("hi"));

fn takes_long_type(f: Thunk) {
    // --snip--
}

fn returns_long_type() -> Thunk {
    // --snip--
}
```

类型别名也常用于减少 `Result<T, E>` 的冗余，例如下面的代码：

```rust
use std::io::Error;
use std::fmt;

pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize, Error>;
    fn flush(&mut self) -> Result<(), Error>;

    fn write_all(&mut self, buf: &[u8]) -> Result<(), Error>;
    fn write_fmt(&mut self, fmt: fmt::Arguments) -> Result<(), Error>;
}
```

上述代码重复出现了 `Result<..., Error>`。因此，在 `std::io` 中，有了此类型的别名：

```rust
type Result<T> = Result<T, std::io::Error>;
```

这样，代码就能够简化为：

```rust
pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;

    fn write_all(&mut self, buf: &[u8]) -> Result<()>;
    fn write_fmt(&mut self, fmt: Arguments) -> Result<()>;
}
```

## Never Type

Rust 还有一个名为 `!` 的类型，它表示空类型（empty type）。Rust 更喜欢称其为 Never Type，因为它反映了一个函数绝不会返回：

```rust
fn bar() => ! {
    // --snip--
}
```

返回 never 的函数被称为**发散函数（diverging functions）**。

## 动态大小类型和 `Sized`

动态大小类型（Dynamically Sized Types），又简称为 DST，这种类型描述了只有在运行时才能知道大小的类型。

`str` 就是一个 DST，这意味着我们无法创建一个类型为 `str` 的变量，也不能接受一个类型为 `str` 的参数。下面的代码就无法工作：

```rust
let s1: str = "Hello there!";
let s2: str = "How's it going?";
```

如果上述代码能工作，则 Rust 应当知道 `s1` 的长度为 12 字节，而 `s2` 的长度为 15 字节。而二者类型又都是 `str`，那么 `str` 这个类型是 12 字节还是 15 字节？

因此，我们修正 `s1` 和 `s2` 的类型为 `&str`。`&str` 的大小是已知的，它包含有两部分：

1. `str` 的地址。
2. `str` 的长度。

我们可以结合 `str` 和各类 pointer：`Box<str>`、`Rc<T>` 等等，来达到运行时内存分配的目的。

为了能与 DST 工作，Rust 有一个特殊的 trait 来决定某个类型是否能在编译期确定大小：`Sized` trait。这个 trait 为编译期能够确定大小的类型自动实现了一切，Rust 会为每个泛型函数隐式地绑定 `Sized`。例如，下面的泛型函数：

```rust
fn generic<T>(t: T) {
    // --snip--
}
```

会被转换为：

```rust
fn generic<T: Sized>(t: T) {
    // --snip--
}
```

默认情况下，泛型函数只会在泛型为已知大小时通过编译，但是，通过下面的语法，你也可以放宽这条限制：

```rust
fn generic<T: ?Sized>(t: &T) {
    // --snip--
}
```

`T: ?Sized` 可以读作 “`T` 可能，也或者不可能是 `Size` （大小已知）”。同时，也要注意，我们将参数 `t` 从头 `T` 转换为 `&T`，这是因为，由于 `T` 现在可能是未知大小，我们需要通过一些 pointer 来使用它，这里我们选择了引用。

