# Expression（表达式）

在 C 语言中，表达式（expression）和语句（statement）是严格区分的

```c
// expression
5 * (fahr - 32) / 9
  
// statement
for (; begin != end; ++begin) {
  if (*begin === target)
    break;
}
```

Rust 则是一门表达式语言（就像 Lisp 那样），表达式在 Rust 中可以做一切工作：`if` 在 Rust 中可以返回值：

```rust
let status = 
  if cpu.temperature <= MAX_TEMP {
    HttpStatus::Ok
  } else {
    HttpStatus::ServerError  
  }
```

## 声明

函数中声明的函数不能访问外部函数的局部变量：

```rust
use std::io;
use tsd::com::Ordering;

fn show_files() -> io::Result<()> {
  let mut v = vec![];
  // ...
  fn com_by_timestamp_then_name(a: &FileInfo, b: &FileInfo) -> Ordering {
    a.timestamp.cmp(&b.timestamp)
    	.reverse()
    	.then(a.path.com(&b.path))
  }
  
  v.sort_by(cmp_by_timestamp_then_name);
  // ...
}
```

内部定义的函数 `com_by_timestamp_then_name` 并不能访问外部函数 `show_files` 中的局部变量 `v`。 

 ## If and Match

If 形式：

```rust
if condition1 {
  block1
}	else if condition2 {
  block2
} else if condition3 {
  block3
}
```

Rust 提供的模式匹配（Pattern Matching）很强大：

```rust
match value {
  pattern => expr
  ...
}
```

If let 则可以作为 match 的简写模式：

```rust
if let pattern = expr {
  block1
} else {
  block2
}
```

当表达式 `expr` 匹配到对应的 `pettern`，就会执行 `block1`。

## Why Rust Has `loop`

Rust 会按照下面的方式分析程序的控制流：

- Rust 会检查函数中每一条路径是否返回了预期的数据类型，为此，Rust 需要知道它是否能够到达函数终点。
- Rust 会检查每个局部变量是否使用或者未被初始化，这需要检查函数的每一条路径是否存在一个未被初始化就使用了的变量。
- 对于不可达的代码，Rust 会告警，代码不可达值得是函数中没有任何一条路径能到达这段代码。

这是一种 [flow-sensitive](https://www.wikiwand.com/en/Data-flow_analysis#/Sensitivities) 分析方式，这些规则需要编程语言在简单和智慧上做权衡。简单意味着开发者能够理解当前编译器在说什么，而智慧，则意味着编译器能够减少错误识别一个安全程序的可能。Rust 选择了简单，它的 flow-sensitive 分析不会检查循环条件，取而代之的是，程序中任何的循环条件，要么是 true，要么是 false，这就造成了 Rust 会拒绝一些安全的程序：

```rust
fn wait_for_process(process: &mut Process) -> i32 {
  while true {
    if process.wait() {
      return process.exit_code();
    }
  }
} // error: not all control paths return a value
```

这个函数不是所有路径都返回了一个 `i32` 类型的值，因此会被 Rust 编译器拒绝。为此，Rust 提供了 `loop` 表达式让开发者更清晰地阐明意图：

```rust
fn wait_for_process(process: &mut Process) -> i32 {
  loop {
    if process.wait() {
      return process.exit_code();
    }
  }
} // error: not all control paths return a value
```

另外，一些函数是永远不会返回结果的（只会被突然中断），这些函数被称为是 divergent function（发散函数）。Rust 使用 `!` 来表示一个函数是一个发散函数，它不会返回结果：

```rust
fn server_forever(socket: ServerSocket, handler: ServerHandler) -> ! {
  socket.listen();
  loop {
    let s = socket.accept();
    handler.handle(s);
  }
}
```



