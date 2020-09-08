# Lifetime

在 Rust 中，每个引用都是有寿命的，它描述了引用的有效范围。例如下面的代码中，当 `x` 离开了作用域之后，其引用 `r` 由于是定义在更靠外的作用于的，因此仍然有效，但指向了不可靠的位置，造成了悬摆引用：

```rust
{
    let r;

    {
        let x = 5;
        r = &x;
    }

    println!("r: {}", r);
}
```

这段代码将报错：

```
error: `x` does not live long enough
   |
6  |         r = &x;
   |              - borrow occurs here
7  |     }
   |     ^ `x` dropped here while still borrowed
...
10 | }
   | - borrowed value needs to live until here
```

为此，Rust 引出了寿命（lifetime）的概念。，

## Borrow Checker

```rust
{
    let r;         // -------+-- 'a
                   //        |
    {              //        |
        let x = 5; // -+-----+-- 'b
        r = &x;    //  |     |
    }              // -+     |
                   //        |
    println!("r: {}", r); // |
                   //        |
                   // -------+
}
```

我们用 `'a` 标识 `r` 的寿命，用 `'b` 标识 `x` 的寿命，可以看到，**引用比引用内容存活得久**，这会造成悬摆引用。

再看到下面的代码：

```rust
{
    let x = 5;            // -----+-- 'b
                          //      |
    let r = &x;           // --+--+-- 'a
                          //   |  |
    println!("r: {}", r); //   |  |
                          // --+  |
}                         // -----+
```

这里，引用内容存活时间足够，避免了悬摆引用。

## 在函数定义中声明 lifetime

```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}

fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

这段代码将不能通过编译：

```rust
error[E0106]: missing lifetime specifier
   |
1  | fn longest(x: &str, y: &str) -> &str {
   |                                 ^ expected lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but the
   signature does not say whether it is borrowed from `x` or `y`
```

Rust 的 Borrow Checker 无法知道引用 `result` 到底是 `x` 还是 `y`，因此无法通过上面的比较法则，知悉引用 `result` 是否会比它所引用的内容 `string1` 或者 `string2` 存活地更久，如果是的话，那势必又将造成悬摆引用，Rust 认为这是不安全的。

为此，Rust 引入了 lifetime 的概念，允许开发者声明引用的寿命。下面的代码中，声明了返回的引用将会和 `x`、`y` 存活一样久，这样 Rust 就知道了，`longest` 返回的引用将会和引用的内容存活一样久，不会出现悬摆引用：

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

而对于下面这段代码，当 `string2` 离开作用域后，由于我们声明了 `result` 会存活与其参数一样久，因此，`result` 也离开了作用域，被销毁，避免了悬摆引用出现。

```rust
fn main() {
    let string1 = String::from("long string is long");

    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    }
}
```

再看到下面的代码，`string2` 离开了作用域，`result` 在作用域外部仍被使用，可能造成悬摆引用。因此无法通过编译。

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

```
error: `string2` does not live long enough
   |
6  |         result = longest(string1.as_str(), string2.as_str());
   |                                            ------- borrow occurs here
7  |     }
   |     ^ `string2` dropped here while still borrowed
8  |     println!("The longest string is {}", result);
9  | }
   | - borrowed value needs to live until here
```

## 结构体中的 lifetime 声明

如果我们的结构体属性是一个引用，就可能出现：当属性引用的内容被销毁了，由于结构体没有被销毁，其属性的引用就会变成悬摆引用，因此，Rust 也支持为结构体属性声明 lifetime。下面的代码中，我们就告诉了 Rust， `ImportantExcerpt` 实例不会比 `part` 活的久，当 `part` 引用的内容销毁后，结构体也会被销毁：

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.')
        .next()
        .expect("Could not find a '.'");
    let i = ImportantExcerpt { part: first_sentence };
}
```

## 省略 lifetime 声明

当然，为所有的引用声明 lifetime 也太啰嗦了。编译时，Rust 遵从以下规则隐式声明 lifetime：

1. 为各个参数声明一个 lifetime，`fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`。
2. 如果入参中只有一个声明了 lifetime，则函数输出与此拥有一致的 lifetime：`fn foo<'a>(x: &'a i32) -> &'a i32`。
3. 如果多个入参声明了 lifetime，并且其中之一是 `&self` 或 `&mut self`，那么输出会与 `self` 拥有一致的 lifetime。


这些规则让我们在声明函数式可以省略 lifetime 声明。

下面这段代码：

```rust
fn first_word(s: &str) -> &str {
```

根据第一条规则，会被翻译成，保证各个引用有 lifetime：

```rust
fn first_word<'a>(s: &'a str) -> &str {
```

根据第二条规则，进而被翻译成，保证返回的引用存活的与参数引用一样久：

```rust
fn first_word<'a>(s: &'a str) -> &'a str {
```

而下面这段代码：

```rust
fn longest(x: &str, y: &str) -> &str {
```

根据第一条规则，会被翻译为，保证各个引用有 lifetime：

```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str {
```

而不满足第二、三条规则。

下面这段代码：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

根据第一条规则，会被翻译为，保证各个引用有 lifetime：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&'a self, announcement: &'b str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

再根据第三条规则，会被翻译为，保证返回的引用存活的与当前对象引用一样久：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&'a self, announcement: &'b str) -> &'a str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

## Static Lifetime

```rust
let s: &'static str = "I have a static lifetime.";
```

`'static` 声明的 lifetime 是整个程序的寿命，避免滥用。

## Demo

下面的这个 Demo 结合了泛型，lifetime，trait bounds：

```rust
use std:fmt::Display;

fn longest_with_an_announcement<'a, T>(
  x: &'a str,
  y: &'a str,
  ann: T,
) -> &'a str
where T: Display {
  println!("Announcement! {}", ann);
  if x.len() > y.len() {
    x
  } else {
    y
  }
}
```

