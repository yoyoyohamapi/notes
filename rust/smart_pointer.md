# Smart Pointer

smart pointer（智慧指针）指的是**具备元信息和一些功能的**结构体，因此，其不只能像指针一样指向某个数据，还具备更多的功能。

在 Rust 中，smart pointer 都实现了 `Deref` 和 `Drop` trait。Rust 有下面几种 smart pointer：

- `Box<T>`：提供了堆上分配值的能力（变量大小不确定，只能在堆上开辟空间）。
- `Rc<T>`：通过引用计数，支持了多所有权。
- `RefCell<T>`：让借用规则（borrow rules）作用在执行期而不是编译期。

## `Box<T>`

### 问题：递归类型

我们定义一个 fp 语言中常见的 cons list：

```rust
enum List {
    Cons(i32, List),
    Nil,
}
```

在 main 函数中，我们创建一个 cons list：

```rust
use List::{Cons, Nil};

fn main() {
    let list = Cons(1, Cons(2, Cons(3, Nil)));
}
```

编译这段代码，将出现错误：

```
error[E0072]: recursive type `List` has infinite size
 --> src/main.rs:1:1
  |
1 | enum List {
  | ^^^^^^^^^ recursive type has infinite size
2 |     Cons(i32, List),
  |               ----- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to
  make `List` representable
```

错误信息显示了，`List` 有无限的大小，因为在其定义中，递归的引用了自己的类型，Rust 无法在编译期间确定为其分配多少的大小。下面的图更加清晰：

![](https://doc.rust-lang.org/book/second-edition/img/trpl15-01.svg)

### 使用 `Box<T>` 为递归类型确定大小

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let list = Cons(1,
    	Box::new(Cons(2,
        	Box::new(Cons(3, 
        		Box::new(Nil))))));
}
```

由于 `Box<T>` 是一个指针类型，因此 Rust 在编译器就能够确定内存分配：

![](https://doc.rust-lang.org/book/second-edition/img/trpl15-02.svg)

现在，`Cons` 所需要的大小就是 `i32` 大小加上 Box 指针的大小。`Box<T>` 将指向下一个在堆上进行内存分配的 `List`。

## `Deref` Trait：允许指针被当做引用使用

实现 `Deref` Trait，允许我们自定义**解引用**操作符 `*` 的行为。实现 `Deref` Trait，能让 smart pointer 被当做普通引用进行使用。

### 解引用 `*`

```rust
fn main() {
    let x = 5;
    let y = &x;
    
    assert_eq!(5, x);
    assert_eq!(5, y);
}
```

执行这段代码，将会报错：

```
error[E0277]: the trait bound `{integer}: std::cmp::PartialEq<&{integer}>` is
not satisfied
 --> src/main.rs:6:5
  |
6 |     assert_eq!(5, y);
  |     ^^^^^^^^^^^^^^^^^ can't compare `{integer}` with `&{integer}`
  |
  = help: the trait `std::cmp::PartialEq<&{integer}>` is not implemented for
  `{integer}`
```

这是因为 `y` 是引用类型，而 `5` 是 i32 类型，因此，我们需要通过使用 **`*` ** 操作符来获得 `y` 所引用的内容：

```rust
fn main() {
    let x = 5;
    let y = &x;
    
    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

由于 `Box<T>` 也实现了 `Deref` Trait，因此可以被当做一般引用进行解引用操作：

```rust
fn main() {
    let x = 5;
    let y = Box::new(x);
    
    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

### 自定义我们的 Box 并实现 `Deref` Trait

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type target = T;
    
    fn deref(&self) -> &T {
        &self.0
    }
}

fn main() {
    let x = 5;
    let y = MyBox::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

### 函数和方法的隐式 Deref coercions

假定我们有一个 `hello` 函数，它接受一个 `&str` 类型：

```rust
fn hello(name: &str) {
    println!("Hello, {}", name);
}
```

Deref coercions 能帮助我们传入一个引用类型的 `MyBox<String>` 变量调用 `hello` 函数：

```rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&m);
}
```

而不需要显示地解引用再进行 string slice 操作：

```rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&(*m)[..]);
}
```

### Deref coerion 与可变性的交互

通过实现 `Deref` trait，可以重载 `*` 在不可变引用上的使用。Rust 也提供了 `DerefMut` trait 来实现可变引用上 `*` 的重载。

下面三种情形下，Rust 会进行 deref coerion：

- From `&T` to `&U` when `T: Deref<Target=U>`
- From `&mut T` to `&mut U` when `T: DerefMut<Target=U>`
- From `&mut T` to `&U` when `T: Deref<Target=U>`

第三个情形要单独说明下，**可变引用转换为一个不可变引用**是安全的。反之则不然，不可变引用决不能转换为一个可变引用。因为当新的可变引用出现时，Rust 无法保证此时数据只有一个可变引用，这打破了借用规则。

## `Drop` Trait：定义清理工作

`Drop` trait 能够帮助自定义 value 离开作用域时应当执行的工作。`Box<T>` 自定义了 `Drop` trait 来实现 box 指向的堆的空间清理。

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("my stuff") };
    let d = CustomSmartPointer { data: String::from("other stuff") };
    println!("CustomSmartPointer created.");
}
```

运行程序，将输出：

```
CustomSmartPointers created.
Dropping CustomSmartPointer with data `other stuff`!
Dropping CustomSmartPointer with data `my stuff`!
```

### 提前 Drop Value

使用 `std::mem::drop` 可以提前 drop value：

```rust
fn main() {
    let c = CustomSmartPointer { data: String::from("some data") };
    println!("CustomSmartPointer created.");
    drop(c);
    println!("CustomSmartPointer dropped before the end of main.");
}
```

