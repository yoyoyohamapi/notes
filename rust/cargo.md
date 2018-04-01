# Cargo

cargo 可以进行不同环境的构建：

```bash
$ cargo build
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
$ cargo build --release
    Finished release [optimized] target(s) in 0.0 secs
```

`opt-level` 指定了 Rust 的代码优化级别，有 0、1、2、3 级别可以设置，优化等级越高，编译事件越长：

Cargo.toml：

```toml
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```

## 发布 Crate 到 Crates.io

### 使用 \`\\\\\\` 进行功能注释  

使用 `\\\` 进行注释，可以在注释里面撰写 Markdown：

```rust
/// Adds one to the number given.
///
/// # Examples
///
/// ```
/// let five = 5;
///
/// assert_eq!(6, my_crate::add_one(5));
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

通过 `cargo doc --open` 我们能生成注释，并在网页端进行预览：

![Rendered HTML documentation for the `add_one` function of `my_crate`](https://doc.rust-lang.org/book/second-edition/img/trpl14-01.png)

注释需要声明下面三个部分：

- **Panic**：在什么场景下函数会出现 panic。
- **Error**：如果函数返回 `Result` ，需要阐明会包含哪些错误。
- **Safety**：如果函数调用是 `unsafe` 的，需要在文档中说明为什么该函数是不安全的，以及用户的注意事项。

文档也是可以被测试的：

```
   Doc-tests my_crate

running 1 test
test src/lib.rs - add_one (line 5) ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

### 使用 `\\!` 进行模块注释

```rust
//! # My Crate
//!
//! `my_crate` is a collection of utilities to make performing certain
//! calculations more convenient.

/// Adds one to the number given.
// --snip--
```

![](https://doc.rust-lang.org/book/second-edition/img/trpl14-02.png)

## 发布

1. **编辑 toml**：

   ```toml
   [package]
   name = "guessing_game"
   version = "0.1.0"
   authors = ["Your Name <you@example.com>"]
   description = "A fun game where you guess what number the computer has chosen."
   license = "MIT OR Apache-2.0"

   [dependencies]
   ```


2. **执行发布**：

   ```
   $ cargo publish
    Updating registry `https://github.com/rust-lang/crates.io-index`
   Packaging guessing_game v0.1.0 (file:///projects/guessing_game)
   Verifying guessing_game v0.1.0 (file:///projects/guessing_game)
   Compiling guessing_game v0.1.0
   (file:///projects/guessing_game/target/package/guessing_game-0.1.0)
    Finished dev [unoptimized + debuginfo] target(s) in 0.19 secs
   Uploading guessing_game v0.1.0 (file:///projects/guessing_game)
   ```

   如何打版本号，可以参考：[[Semantic Versioning rules](http://semver.org/)](https://semver.org/lang/zh-CN/)

### 删除

从 Cargo.io 上删除版本：

```
cargo yank --vers 1.0.1
```

这避免了有用户在新的项目中使用 1.0.1 这个版本。

yank 操作也是可以撤销的：

```
cargo yank --vers 1.0.1 --undo
```

## Workspace

Cargo 通过 Workspace 来管理多个 package，Cargo.toml：

```toml
[workspace]

members = [
    "adder",
    "add-one",
]
```

通过 `cargo new` 创建我们的 package：

```
$ cargo new add
	Created library `add` project
$ cargo new add-one
     Created library `add-one` project
```

生成的目录如下：

```
├── Cargo.lock
├── Cargo.toml
├── add-one
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
├── adder
│   ├── Cargo.toml
│   └── src
│       └── main.rs
└── target
```

add-one 这个库中定义了一个 `add_one` 函数，add-one/src/lib.rs：

```rust
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

adder 可以在其目录下的 Cargo.toml 中声明依赖，就能使用该函数。adder/Cargo.toml：

```toml
[dependencies]

add-one = { path = "../add-one" }
```

adder/src/main.rs：

```rust
extern crate add_one;

fn main() {
    let num = 10;
    println!("Hello, world! {} plus one is {}!", num, add_one::add_one(num));
}
```

通过参数 `p` ，可以指定需要运行那个 package：

```
cargo run -p adder
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/adder`
Hello, world! 10 plus one is 11!
```

## 从 Cargo.io 获取 package

```
$ cargo install ripgrep
Updating registry `https://github.com/rust-lang/crates.io-index`
 Downloading ripgrep v0.3.2
 --snip--
   Compiling ripgrep v0.3.2
    Finished release [optimized + debuginfo] target(s) in 97.91 secs
  Installing ~/.cargo/bin/rg
```

## 自定义命令

通过 `cargo --list` 可以查看当前有哪些自定义命令，存在于 `$PATH` 中的 `cargo-something` 地址将可以通过 `cargo something` 执行。ß