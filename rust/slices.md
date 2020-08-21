# Slices

另一个不具有所有权（ownership）的数据类型是 slice。

```rust
let s = String::from("hello world");

let hello = &s[0..5];
let world = &s[6..11];
```

![](https://doc.rust-lang.org/book/second-edition/img/trpl04-06.svg)在看到下面的代码：

```rust
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();
    
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    
    &s[..]
}

fn main() {
    let s = String::from("hello world");

    let word = first_word(&s);

    s.clear(); // Error!
}
```

这里将会报错:

```
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
 --> src/main.rs:6:5
  |
4 |     let word = first_word(&s);
  |                            - immutable borrow occurs here
5 |
6 |     s.clear(); // Error!
  |     ^ mutable borrow occurs here
7 | }
  | - immutable borrow ends here
```

这是因为，此时 `first_word`拥有了一个 `s` 的**不可变引用**，而 `s.clear()` 则会截断 `s`，因此它就会需要一个 `s` 的可变引用，这与借用规则相悖，因此报错。

## 字符串字面量

```rust
fn main() {
    let my_string = String::from("hello world");

    // first_word works on slices of `String`s
    let word = first_word(&my_string[..]);

    let my_string_literal = "hello world";

    // first_word works on slices of string literals
    let word = first_word(&my_string_literal[..]);

    // 由于字符串字面量已经是 slice 了，所以也可以这么用
    let word = first_word(my_string_literal);
}

```

