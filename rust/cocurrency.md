# Concurrency（并发）

Rust 将它的并发控制称为 fearless cocurrency，因为通过所有权机制以及类型检测，并发错误将发生在**编译期**，而不是**运行时**，因此，在 Rust 中，你可以无畏并发。

##  使用线程来同时运行代码

使用线程为程序带来性能收益，但也提高了程序的复杂性，由线程引起的问题大致有：

* 竞态问题（race condition）：线程间以不一致的顺序访问数据或者资源。
* 死锁（deadlock）：线程都在等着对方释放资源。
* bug 定位：由于多线程运行的多样性，一些 bug 只会在某种场景下出现，难于定位和复现。

## `spawn` ：创建线程

`spawn` 接收一个闭包作为参数，闭包中定义了**想要在新的线程中执行的代码**：

```rust
use std::thread;
use std::time:Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });
    
    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

当主线程完成时，子线程将不再继续，所以你看到的输出可能是：

```
hi number 1 from the main thread!
hi number 2 from the main thread!
hi number 1 from the spawned thread!
hi number 3 from the main thread!
hi number 2 from the spawned thread!
hi number 4 from the main thread!
hi number 3 from the spawned thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
hi number 6 from the spawned thread!
hi number 7 from the spawned thread!
hi number 8 from the spawned thread!
hi number 9 from the spawned thread!
```

## `join`：等待所有线程执行完毕

`spawn` 方法将返回一个线程句柄，它是 `JoinHandle` 类型，调用该类型的 `join` 方法，程序会保证**子线程在主线程结束前完成**。

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });
    
    handle.join().unwrap();
    
    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

此时，主线程将等到子线程执行完，才会执行：

```
hi number 1 from the spawned thread!
hi number 2 from the spawned thread!
hi number 3 from the spawned thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
hi number 6 from the spawned thread!
hi number 7 from the spawned thread!
hi number 8 from the spawned thread!
hi number 9 from the spawned thread!
hi number 1 from the main thread!
hi number 2 from the main thread!
hi number 3 from the main thread!
hi number 4 from the main thread!
```

## `move`：转交所有权

假定我们新线程的闭包想要试用外部环境的数据：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];
    
    let handle = thread::spawn(|| {
        println!("Here's a vector: {:?}", v);
    });
    
    handle.join().un_wrap();
}
```

编译代码， 却报错：

```
error[E0373]: closure may outlive the current function, but it borrows `v`,
which is owned by the current function
 --> src/main.rs:6:32
  |
6 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `v`
7 |         println!("Here's a vector: {:?}", v);
  |                                           - `v` is borrowed here
  |
help: to force the closure to take ownership of `v` (and any other referenced
variables), use the `move` keyword
  |
6 |     let handle = thread::spawn(move || {
  |                                ^^^^^^^
```

这是因为，Rust 会推测如何捕获 `v`，由于 `println!` 只需要 `v` 的引用，因此闭包尝试去借用 `v`。但是，Rust 无法知道子线程的存活时间，因此也就无法知晓 `v` 的有效期应当维持多久。

例如，我们在主线程直接 drop 掉 `v`：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];
    
    let handle = thread::spawn(|| {
        print!("Here's a vector: {:?}", v);
    });
    
    drop(v);
    
    handle.join().unwrap();
}
```

当子线程创建后（未必开始运行），主线程就立即清除了 `v`，当子线程开始运行时，`v` 也就不再有效了。因此，仅仅是借用 value，并不安全。

Rust 因此提供了 `move` 这个关键字来让闭包直接获得 value 的所有权：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];
    
    let handle = thread::spawn(move || {
        println!("Here's a vector: {:?}", v);
    });
    
    handle.join().unwrap();
}
```

