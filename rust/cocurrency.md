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

##  线程间的消息传递

在 Rust 中，线程间的通信是通过 **channel** 完成的，它类似于一条流动的河，河上的床承载了数据，它包含有两个部分：

- **transmitter**：位于 channel 上游，负责将数据放入 channel。
- **receiver**：位于 channel 下游，负责接收数据。

```rust
use std::sync::mpsc;

fn main() {
    let (tx, ty) = mpsc::channel();
}
```

`mpsc` 是 multiple producer，single consumer 的缩写，亦即**多生产者，单一消费者**，该模型可以假想为多个河流最后并入一条河流，供消费者消费。在细化一点例子，让子线程和主线程通信：

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });
    
    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

运行这段代码，将输出：

```
Got: hi
```

### 所有权

假如，线程发送数据后，仍会使用到数据。这个情况是危险的，因为再次使用该数据前，接收线程可能会修改甚至 drop 掉该数据。因此，`send` 方法将取走数据的所有权，以防代码再次使用该数据：

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
        println!("val is {}", val);
    });
    
    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

```
error[E0382]: use of moved value: `val`
  --> src/main.rs:10:31
   |
9  |         tx.send(val).unwrap();
   |                 --- value moved here
10 |         println!("val is {}", val);
   |                               ^^^ value used here after move
   |
   = note: move occurs because `val` has type `std::string::String`, which does
not implement the `Copy` trait
```

## 迭代接收消息

```rust
use std::thread;
use std::sync::mpsc;
use std::item::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread")
        ];
        
        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration:;from_secs(1));
        }
    });
    
    for received in rx {
        println!("Got: {}", received);
    }
}
```

```
Got: hi
Got: from
Got: the
Got: thread
```

### 创建多个生产者

使用 `mpsc::Sender::clone` 方法能为**同一个** receiver 再创建一个 transmitter：

```rust
let (tx, rx) = mpsc::channel();

let tx1 = mpsc::Sender::clone(&tx);
thread::spawn(move || {
    let vals = vec![
        String::from("hi"),
        String::from("from"),
        String::from("the"),
        String::from("thread"),
    ];

    for val in vals {
        tx1.send(val).unwrap();
        thread::sleep(Duration::from_secs(1));
    }
});

thread::spawn(move || {
    let vals = vec![
        String::from("more"),
        String::from("messages"),
        String::from("for"),
        String::from("you"),
    ];

    for val in vals {
        tx.send(val).unwrap();
        thread::sleep(Duration::from_secs(1));
    }
});

for received in rx {
    println!("Got: {}", received);
}
```

你**可能**看到的输出为：

```
Got: hi
Got: more
Got: from
Got: messages
Got: for
Got: the
Got: thread
Got: you
```

## 共享状态的并发

### 互斥

互斥性借由**锁**系统来保证多个线程访问同一个数据时的安全：

* 使用数据前需要上锁
* 数据使用完要释放锁

如果数据使用完未释放锁，就会造成**死锁**，即各个线程等待数据释放锁。

### `Mutex<T>` API

Rust 专门提供了服务于互斥问题的 API：

```rust
use std::sycn::Mutex;

fn main() {
    let m = Mutex::new(5);
    
    {
        let mut num = m.lock().unwrap();
        
        *num = 6;
    }
    
    println!("m = {:?}", m);
}
```

要想使用互斥量，我们先要通过 `lock` 方法对数据上锁，如果此时有其他线程正在使用该数据，程序将 panic。`lock` 方法将返回一个 `MutexGuard` smart pointer，这个 smart pointer 实现了 `Deref` trait，让我们能够使用 `*` 进行解引用获得数据；也实现了 `Drop` trait，当 `MutexGuard` 离开作用域时，释放互斥量的锁。

### 多线程间共享 `Mutex<T>`

```rust
use std::sycn::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);
    let mut handlers = vec![];
    
    for _ in 0..10 {
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            
            *num += 1;
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("Result: {}", *counter.lock().unwrap());
}
```

这段代码意图很简单，各个线程将依次增减计数量 `counter`，但是它无法通过编译：

```
error[E0382]: capture of moved value: `counter`
  --> src/main.rs:10:27
   |
9  |         let handle = thread::spawn(move || {
   |                                    ------- value moved (into closure) here
10 |             let mut num = counter.lock().unwrap();
   |                           ^^^^^^^ value captured here after move
   |
   = note: move occurs because `counter` has type `std::sync::Mutex<i32>`,
   which does not implement the `Copy` trait

error[E0382]: use of moved value: `counter`
  --> src/main.rs:21:29
   |
9  |         let handle = thread::spawn(move || {
   |                                    ------- value moved (into closure) here
...
21 |     println!("Result: {}", *counter.lock().unwrap());
   |                             ^^^^^^^ value used here after move
   |
   = note: move occurs because `counter` has type `std::sync::Mutex<i32>`,
   which does not implement the `Copy` trait

error: aborting due to 2 previous errors
```

从报错我们可以看到，`counter` 的所有权被第一个线程占有了，因此我们考虑使用上一章中的 `Rc<T>` 来实现数据的多个所有权：

```rust
use std::rc::Rc;
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Rc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Rc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

事与愿违，仍然无法通过编译：

```
error[E0277]: the trait bound `std::rc::Rc<std::sync::Mutex<i32>>:
std::marker::Send` is not satisfied in `[closure@src/main.rs:11:36:
15:10
counter:std::rc::Rc<std::sync::Mutex<i32>>]`
  --> src/main.rs:11:22
   |
11 |         let handle = thread::spawn(move || {
   |                      ^^^^^^^^^^^^^ `std::rc::Rc<std::sync::Mutex<i32>>`
cannot be sent between threads safely
   |
   = help: within `[closure@src/main.rs:11:36: 15:10
counter:std::rc::Rc<std::sync::Mutex<i32>>]`, the trait `std::marker::Send` is
not implemented for `std::rc::Rc<std::sync::Mutex<i32>>`
   = note: required because it appears within the type
`[closure@src/main.rs:11:36: 15:10
counter:std::rc::Rc<std::sync::Mutex<i32>>]`
   = note: required by `std::thread::spawn`
```

因为 `Rc<T>` 并没有为多线程做过优化，因此，不能保证修改引用计数时，会被其他线程所打断。

### `Arc<T>`：原子的引用计数

幸运的是，Rust 为我们提供了线程安全的 `Arc<T>`，来让多个线程共享数据的所有权：

```rust
use std::sync::{Mutex, Arc};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

终于，程序输出为：

```
Result: 10
```

### `RefCell<T>/Rc<T>` <=> `Mutex<T>/Arc<T>`

`counter` 是不可变的，但在线程运行时，我们却获得了其可变引用，这意味着，类似 `RefCell<T>` ，`Mutex<T>` 提供了**内在可变性**。

## 并发特性的扩展

Rust 本身提供了有限的并发特性，但通过实现 `Send` 和 `Sync` trait，我们可以扩展并发特性。

### Trait：`Send`

`Send` marker trait 指出了，实现了 `Send ` trait 的类型可以在各个线程中传递。但是 `Rc<T>` 是个例外，这是因为，如果我们尝试传递克隆对象的所有权到其他线程，那么其他线程可能**同时更新引用计数**。因此，`Rc<T>` 是服务于**单线程**的。几乎所有原始类型都是可 `Send` 的，`Send` 类型进行复合，也是 `Send` 类型的。

### Trait： `Sync`

`Sync` marker trait 指出了，实现了 `Sync` trait 的类型可以被多个线程引用。换言之，如果 `&T` 是 `Send`，那么类型 `T` 是 `Sync` 的，意味着**引用可以被安全地传递给其他线程**。Rust 中原始类型及其复合类型都是 `Sync` 的。

`Rc<T>` 仍然不是 `Sync` 的，理由如上。`RefCell<T>` 同样不是 `Sync` 的，因为其运行时满足借用规则不是线程安全的。