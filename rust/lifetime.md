# Lifetime

Rust 使用 Lifetime 机制来避免**悬摆引用**，提高代码安全和健壮性：

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

当 `x` 离开作用域被清空时，其引用 `r` 不知指向何处，是非常不安全的。

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

我们用 `'a` 标识 `r` 的生命期，用 `'b` 标识 `x` 的生命期，可以看到，**引用比引用内容存活得久**，这会造成悬摆引用。

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

`result` 是一个引用，上述代码无法使用 Borrow Checker 比较引用（`result`）和引用内容（`string1`, `string2`）的生命期，因此被 Rust 认为是不安全的。

为了解决这个问题，我们在函数签名中声明 lifetime：

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

现在，函数返回的引用将与其参数拥有同样的生命期，即 `result` 将会和 `string1`、`string2` 存活同样久，当 `string1`、`string2` 被销毁时，引用的生命期也结束了，避免了悬摆引用的产生。

下面这段代码所以也是没问题的：

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

当 `string2` 离开作用域时， `result` 也离开作用域，而被销毁。不存在悬摆引用。

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

这里的 lifetime 声明保证了结构体属性不会比结构体存活更久。

## 省略 lifetime 声明

编译时，Rust 遵从以下规则声明 lifetime：

1. 为各个参数声明一个 lifetime，`fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`。
2. 如果入参中只有一个声明了 lifetime，则函数输出与此拥有一致的 lifetime：`fn foo<'a>(x: &'a i32) -> &'a i32`。
3. 如果多个入参声明了 lifetime，并且其中之一是 `&self` 或 `&mut self`，那么输出会与 `self` 拥有一致的 lifetime。


这些规则让我们在声明函数式可以省略 lifetime 声明。

下面这段代码：

```rust
fn first_word(s: &str) -> &str {
```

根据第一条规则，会被翻译成：

```rust
fn first_word<'a>(s: &'a str) -> &str {
```

根据第二条规则，进而被翻译成：

```rust
fn first_word<'a>(s: &'a str) -> &'a str {
```

而下面这段代码：

```rust
fn longest(x: &str, y: &str) -> &str {
```

根据第一条规则，会被翻译为：

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

根据第一条规则，会被翻译为：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&'a self, announcement: &'b str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

再根据第三条规则，会被翻译为：

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

`'static` 声明的 lifetime 是整个程序的生命期，避免滥用。