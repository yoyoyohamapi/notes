# 引用（Reference）

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

在 Rust 中，使用 & 来创建引用，此时，`s1` 的所用权不会被转移，这样，当 `s` 离开作用域时，其引用的内容不会被 drop：

![](https://doc.rust-lang.org/book/second-edition/img/trpl04-05.svg)

如果尝试改变引用的内容，将报错：

```rust
fn main() {
    let s = String::from("hello");

    change(&s);
}

fn change(some_string: &String) {
    some_string.push_str(", world");
}
```

```
error[E0596]: cannot borrow immutable borrowed content `*some_string` as mutable
 --> error.rs:8:5
  |
7 | fn change(some_string: &String) {
  |                        ------- use `&mut String` here to make mutable
8 |     some_string.push_str(", world");
  |     ^^^^^^^^^^^ cannot borrow as mutable
```

## 可变引用

为了这个，我们可以使用**可变引用**：

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

同一时刻，只能拥有一个可变引用，下面的代码将报错：

```rust
let mut s = String::from("hello");

let r1 = &mut s;
let r2 = &mut s;
```

```
error[E0499]: cannot borrow `s` as mutable more than once at a time
 --> borrow_twice.rs:5:19
  |
4 |     let r1 = &mut s;
  |                   - first mutable borrow occurs here
5 |     let r2 = &mut s;
  |                   ^ second mutable borrow occurs here
6 | }
  | - first borrow ends here
```

Rust 这么做的目的在于，从源头上避免数据竞争（data race）：

1. 两个或多个指针同时指向了相同数据。
2. 其中一个指针被用来写数据。
3. 缺乏机制来同步数据的访问。

借助于作用域，我们可以创建多个可变引用：

```rust
let mut s = String::from("hello");

{
    let r1 = &mut s;

} // r1 已经离开了作用域，因此我们再度创建 s 的可变引用是没用问题的。

let r2 = &mut s;
``` 

类似地，可以创建多个不可变引用，但是在创建不可变引用之后，不能再创建可变引用，这也是要从源头上遏制数据竞争：

```rust
let mut s = String::from("hello");

let r1 = &s; // no problem
let r2 = &s; // no problem
let r3 = &mut s; // BIG PROBLEM
```

```
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as
immutable
 --> borrow_thrice.rs:6:19
  |
4 |     let r1 = &s; // no problem
  |               - immutable borrow occurs here
5 |     let r2 = &s; // no problem
6 |     let r3 = &mut s; // BIG PROBLEM
  |                   ^ mutable borrow occurs here
7 | }
  | - immutable borrow ends here
```

## 悬摆引用（Dangling References）

在一些语言中，悬摆指针指的是一个指针会在其指向的内存对象被释放后可以指向另一个内存位置。在 Rust 中，编译器保证了引用绝不会成为悬摆引用，如果我们已经引用了某个数据，那么编译器将保证数据不会先于其引用离开作用域，这样引用就不会悬空、悬摆。

下面这段代码将报错：

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {
    let s = String::from("hello");

    &s // 返回一个字符串的引用，但是因为 s 已经离开了作用域，因此其会被释放，造成了引用悬空
}
```

通过转移所有权来解决这个问题：

```rust
fn no_dangle() -> String {
    let s = String::from("hello");

    s
}
```

## 总结

1. 任何时候，你将拥有下述其一：
    - 一个可变引用
    - 任何数量的不可变引用
2. 引用不能悬空
