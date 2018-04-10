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

## `Rc<T>`：引用计数

多数时候，所有权是容易知晓的：你可以知道哪个变量拥有给定的值。但是有些时候，某个值可能被多个变量拥有。

例如一个图，多个边可能指向同一个顶点，此时，该节点就被这些边所拥有。

Rust 提供了 `Rc<T>` 来完成引用计数。当我们想要程序多个部分都会读取的数据进行堆上的内存分配，我们又无法在编译期确定哪个部分最后消耗数据，就考虑使用 `Rc<T>`。如果我们知道最后消耗数据的代码，我们可以让那部分称为数据的所有者，从而在编译期应用所有权规则即可。

## 问题

假定我们有下面的 3 个 cons list，并且它们的关系为：

![](https://doc.rust-lang.org/book/second-edition/img/trpl15-03.svg)

据此，我们撰写的 Rust 代码就为：

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let a = Cons(5, 
        Box::new(Cons(10, 
    		Box::new(Nil))));
    let b = Cons(3, Box::new(a));
    let c = Cons(4, Box::new(a));
}
```

运行这段代码，无法通过编译：

```
error[E0382]: use of moved value: `a`
  --> src/main.rs:13:30
   |
12 |     let b = Cons(3, Box::new(a));
   |                              - value moved here
13 |     let c = Cons(4, Box::new(a));
   |                              ^ value used here after move
   |
   = note: move occurs because `a` has type `List`, which does not implement
   the `Copy` trait
```

`a` 的所有权已经被转移了，无法再使用 `a` 了。

## 引用计数

```rust
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
}
```

现在，当我们创建 `b` 的时候，不会再夺走 `a` 的所有权，而是 clone 一份 `a` 所持有的 `Rc<T>` （指针）。clone 操作会增加 `a` 的引用计数，当引用计数为 0 时，`a` 将会被清除。

```rust
fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    println!("count after creating a = {}", Rc::strong_count(&a));
    let b = Cons(3, Rc::clone(&a));
    println!("count after creating b = {}", Rc::strong_count(&a));
    {
        let c = Cons(4, Rc::clone(&a));
        println!("count after creating c = {}", Rc::strong_count(&a));
    }
    println!("count after c goes out of scope = {}", Rc::strong_count(&a));
}
```

这段代码将输出：

```
count after creating a = 1
count after creating b = 2
count after creating c = 3
count after c goes out of scope = 2
```

## `RefCell<T>`

Rust 中提供的**内在可变性（interior mutability）** 允许你改变不可变引用指向的数据。这一特性允许你绕过借用规则的限制，修改数据，因此，这个模式在数据内部使用了 `unsafe` 的代码（使用 safe API 包裹了 `unsafe` 的代码，数据因此还是在编译期保持了不可变性）。

内在不变性仅只是遮蔽了编译期的借用规则，但是在运行时，你还是要保证对数据的引用满足借用规则。

### 使用 `RefCell<T>` 保证运行时的借用规则

再回顾下借用规则：

- 任何时候，你可以拥有某个数据任意数目的不可变引用和**一个**可变引用。
- 引用必须有效。

`RefCell<T>` 和 `Box<T>` 都只能拥有数据的一个所有权。

- `Box<T>` 需要在**编译期**满足借用规则。
-  `RefCell<T>` 只需要在**运行时**满足借用规则。
- `Box<T>` 违反借用规则时，编译不通过。
- `RefCell<T>` 违反借用规则时，程序将 `panic!` 并退出。

### 用例：Mock

我们设计一个 LimitTracker，它跟踪当前我们调用次数和调用上限的距离，进行不同的提示：

```rust
pub trait Messenger {
    fn send(&self, msg: &str);
}

pub struct LimitTracker<'a, T: 'a + Messenger> {
    messanger: &a' T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T> where T: Messenger {
    pub fn new() {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }
    
    pub fn set_value(&mut self, value: usize) {
        self.value = value;
        let percentage_of_max = self.value as f64 / self.max as f64;
        
        if percentage_of_max >= 0.75 && percentage_of_max < 0.9 {
            self.messenger.send("Warning: You've used up over 75% of your quota!");
        } else if {
            self.messenger.send("Urgent warning: You've used up over 90% of your quota!");
        } else if {
            self.messenger.send("Error: You are over your quota");
        }
    }
    
}
```

注意到，`Messenger` Trait 中，`send` 的函数签名使用的是**不可变**的 `self` 引用。

下面我们撰写测试代码，创建一个 `MockMessenger` 来测试我们的 `LimitTracker`：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    struct MockMessenger {
        sent_messages: Vec<String>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger { sent_messages: vec![] }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);

        limit_tracker.set_value(80);

        assert_eq!(mock_messenger.sent_messages.len(), 1);
    }
}
```

我们预期程序能输出 `'Warning: You've used up over 75% of your quota!'`。但是，程序无法通过编译：

```
error[E0596]: cannot borrow immutable field `self.sent_messages` as mutable
  --> src/lib.rs:52:13
   |
51 |         fn send(&self, message: &str) {
   |                 ----- use `&mut self` here to make mutable
52 |             self.sent_messages.push(String::from(message));
   |             ^^^^^^^^^^^^^^^^^^ cannot mutably borrow immutable field
```

`MockMessenger` 实现的 `send` 方法，`self` 是不可变引用，但在 `send` 内部，`MockMessenger` 改变了自己的 `sent_messages`。此时，我们可能会想实现 `send(&mut self, message)`，但 `Messenger` 提供的 `send` 函数签名不是如此。

我们这段代码应该是合理的，但因为违反了借用规则，因此被 Rust 拒绝在了编译期。这时，我们可以使用 `RefCell<T>` 来让借用规则延后至**运行时**。

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell:RefCell;
    
    struct MockMessenger {
        sent_messages:: RefCell<Vec<String>>,
    }
    
    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger { 
                sent_messages: RefCell::new(vec![])
            } 
        }
    }
    
    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.borrow_mut().push(String::from(message));
        }
    }
    
    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```

- `borrow_mut()`：在运行时借出数据的可变引用。
- `borrow()`：在运行时借出数据的不可变引用。

### 在运行时遵守借用规则

在使用了 `RefCell<T>` 后，我们需要在**运行时遵守借用规则**：

```rust
impl Messenger for MockMessenger {
    fn send(&self, message: &str) {
         let mut one_borrow = self.sent_messages.borrow_mut();
        let mut two_borrow = self.sent_messages.borrow_mut();

        one_borrow.push(String::from(message));
        two_borrow.push(String::from(message));
    }
}

```

上面代码在运行时没有遵循借用规则，程序将 panic 并且终止：

```
---- tests::it_sends_an_over_75_percent_warning_message stdout ----
    thread 'tests::it_sends_an_over_75_percent_warning_message' panicked at
    'already borrowed: BorrowMutError', src/libcore/result.rs:906:4
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

### 拥有可变数据的多个所有权

通过组合 `RefCell<T>` 和 `Rc<T>` ，还可以实现拥有某个**可变**数据的**多个**所有权。

![](https://doc.rust-lang.org/book/second-edition/img/trpl15-03.svg)



```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil
}

use List::{Cons, Nil};
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let value = Rc::new(RefCell::new(5));
    
    let a = Rc::new(Cons(RefCell::clone(&value)), Rc::new(Nil));
    let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(10)), Rc::clone(&a));
    
    *value.borrow_mut() += 10;
    
    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```

运行程序，将输出：

```
a after = Cons(RefCell { value: 15 }, Nil)
b after = Cons(RefCell { value: 6 }, Cons(RefCell { value: 15 }, Nil))
c after = Cons(RefCell { value: 10 }, Cons(RefCell { value: 15 }, Nil))
```

## 循环引用与 `Weak<T>`

下面这段代码将创建一个循环引用：

```rust
use std::rc::Rc;
use std::cell::RefCell;
use List::{Cons, Nil};

#[derive(Debug)]
enum List {
  Cons(i32, RefCell<Rc<List>>), // 使得链接的 List 能够在运行时拥有多个可变引用
  Nil
}

impl List {
  fn tail(&self) -> Option<&RefCell<Rc<List>>> {
    match *self {
      Cons(_, ref item) => Some(item),
      Nil => None
    }
  }
}

fn main() {
  let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));

  println!("a initial rc count = {}", Rc::strong_count(&a));
  println!("a next item = {:?}", a.tail());

  let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));

  println!("a rc count after b creation = {}", Rc::strong_count(&a));
  println!("b initial rc count = {}", Rc::strong_count(&b));
  println!("b next item = {:?}", b.tail());

  if let Some(link) = a.tail() {
    *link.borrow_mut() = Rc::clone(&b);
  }

  println!("b rc count after changing a = {}", Rc::strong_count(&b));
  println!("a rc count after changing a = {}", Rc::strong_count(&a));
}
```

![](https://doc.rust-lang.org/book/second-edition/img/trpl15-04.svg)

由于 `a`、`b` 的引用计数都不为0，二者就都不会被清空，循环引用因此造成**内存泄露**。如果我们在 `main()` 函数的最后加入：

```rust
 println!("b next item = {:?}", b.tail());
```

那么将无限循环，并导致栈溢出。

### 使用 `Weak<T>` 来防止循环引用

`Rc<T>` 是代表的是**强引用**，只有当引用计数为 0 时，其实例才会被清除。通过 `Rc::downgrade` ，我们可以将强引用 `Rc<T>` 降为**弱引用** `Weak<T>`。`Rc<T>` 使用 `weak_count` 来追踪弱引用的数目。对于 `Rc<T>` 的清理，不需要弱引用数目为 0。

一旦强引用数目为 0，弱引用就会被切断。因此，在使用弱引用的过程中，我们要确定其目前是否存在。这可以通过调用弱引用上的 `upgrade` 方法，该方法会返回一个 `Option<Rc<T>>`。如果 `Rc<T>` 被释放，则返回 `None`，否则返回 `Some(Rc<T>)`。

### 示例：Tree

树是常见的数据结构，我们可以这样定义：

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node {
  value: i32,
  children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
  // 新建叶子节点
  let leaf = Rc::new(Node {
    value: 3,
    children: RefCell::new(vec![])
  });

  // 树干节点
  let branch = Rc::new(Node {
    value: 5,
    children: RefCell::new(vec![Rc::clone(&leaf)]),
  });
}
```

我们看到，树干节点持有了叶子节点的引用（强引用），我们可能还需要叶子节点也持有树干节点，也就是它父亲节点的引用。为此，我们需要在结构体中添加一个 `parent` 属性用来标识父亲节点。现在，我们需要考虑的就是 `parent` 应当是什么类型：

- `Rc<T>`：此时会存在**循环引用**，`leaf.parent` 指向了 `branch`，而 `branch.children` 指向了 `leaf`，这会让 `strong_count` 不为 0。

换一种方式思考，父亲节点销毁时，其子孙应当被销毁，这意味着父亲节点应当持有子孙节点的**强引用**。子孙节点销毁时，父亲节点不应该销毁，因此子孙只应该持有父亲节点的**弱引用**。

综上，`parent` 的类型应当为 `Weak<T>`：

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf),
    );

    {
        let branch = Rc::new(Node {
            value: 5,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(vec![Rc::clone(&leaf)]),
        });

        *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

        println!(
            "branch strong = {}, weak = {}",
            Rc::strong_count(&branch),
            Rc::weak_count(&branch),
        );

        println!(
            "leaf strong = {}, weak = {}",
            Rc::strong_count(&leaf),
            Rc::weak_count(&leaf),
        );
    }

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf),
    );
}

```



