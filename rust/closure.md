# Closure（闭包）

## 定义一个闭包

```rust
let expensive_closure = |num| {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    num
};
```

## 类型推导

出于下面的几个理由，Rust 会为闭包做类型推导：

- 闭包以变量形式存储，交给库内部使用，而不暴露给外部
- 闭包很简单，并且只会关联一个小范围的上下文

看到下面的代码：

```rust
let example_closure = |x| x;

let s = example_closure(String::from("hello"));
let n = example_closure(5);
```

Rust 已经推导出闭包的参数是字符串类型，因此，运行这段代码会报错：

```
error[E0308]: mismatched types
 --> src/main.rs
  |
  | let n = example_closure(5);
  |                         ^ expected struct `std::string::String`, found
  integral variable
  |
  = note: expected type `std::string::String`
             found type `{integer}`
```

当然，为闭包声明变量类型也是可以的：

```rust
let expensive_closure = |num: u32| -> u32 {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    num
};
```

## 访问外部变量

闭包可以访问其**定义时所在作用域**的变量：

```rust
fn main() {
    let x = 4;

    let equal_to_x = |z| z == x;

    let y = 4;

    assert!(equal_to_x(y));
}
```

如果使用一般函数：

```rust
fn main() {
    let x = 4;

    fn equal_to_x(z: i32) -> bool { z == x }

    let y = 4;

    assert!(equal_to_x(y));
}
```

就会报错：

```rust
error[E0434]: can't capture dynamic environment in a fn item; use the || { ...
} closure form instead
 --> src/main.rs
  |
4 |     fn equal_to_x(z: i32) -> bool { z == x }
  |   
```

