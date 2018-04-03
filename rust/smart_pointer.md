# Smart Pointer

smart pointer（智慧指针）指的是**具备元信息和一些功能的**结构体，因此，其不只能像指针一样指向某个数据，还具备更多的功能。

在 Rust 中，smart pointer 都实现了 `Deref` 和 `Drop` trait。Rust 有下面几种 smart pointer：

- `Box<T>`：提供了堆上分配值的能力（变量大小不确定，只能在堆上开辟空间）。
- `Rc<T>`：通过引用计数，支持了多所有权。
- `RefCell<T>`：让借用规则（borrow rules）作用在执行期而不是编译期。

## `Box<T>`

### 递归类型

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



