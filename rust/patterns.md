# Pattern Matching（模式匹配）

## `match`

在 Rust 中，patterns 无所不在

```
match VALUE {
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
}
```

`match` expressions are required to be *exhaustive*

There’s a particular pattern `_` that will match anything, but never binds to a variable, and so is often used in the last match arm.

## 处处都是模式匹配

### `if let`

```rust
fn main() {
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Some(color) = favorite_color {
        println!("Using your favorite color, {}, as the background", color);
    } else if is_tuesday {
        println!("Tuesday is green day!");
    } else if let Ok(age) = age {
        if age > 30 {
            println!("Using purple as the background color");
        } else {
            println!("Using orange as the background color");
        }
    } else {
        println!("Using blue as the background color");
    }
}
```

### `while let`

```rust
let mut stack = Vec::new();

stack.push(1);
stack.push(2);
stack.push(3);

while let Some(top) = stack.pop() {
    println!("{}", top);
}
```

### `for` 循环

```rust
let v = vec!['a', 'b', 'c'];

for (index, value) in v.iter().enumerate() {
    println!("{} is at index {}", value, index);
}
```

### `let` 语句

Rust 中，`let` 语句也使用了模式匹配：

```rust
let x = 5;
```

该语句满足的模式匹配为：

```rust
let PATTERN = EXPRESSION;
```

### 函数参数

函数参数实际上也使用了模式匹配：

```rust
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({}, {})", x, y);
}

fn main() {
    let point = (3, 5);
    print_coordinates(&point);
}
```

## 可驳（refutable）与不可驳（irrefutable）的模式匹配

Rust 中，有两种模式匹配：

- **irrefutable**：不可反驳的模式匹配，使用 `match` 进行模式匹配时，我们需要声明所有可能的模式，并进行匹配。
- **refutable**：可反驳的模式匹配，我们不需要穷尽所有的模式，如 `if let。`

`let` 语句，函数参数，以及 `for` 循环都是 irrefutable 模式，因为程序无法知道，在某种未匹配到的模式下该如何工作。而 `if let` 和 `while let` 则是 refutable 的，因为我们只需要告诉程序在某种情况该作何反应。

对于一个 irrefutable 的模式，如果我们进行 refutable 的模式匹配，例如：

```rust
let Some(x) = some_option_value;
```

如果 `some_option_value` 是一个 `None` 值，那么它将无法匹配到 `Some(x)`，这意味着这是个 refutable 的模式。而 `let` 语句只接收 irrefutable 模式，因为此时没有任何有效代码来处理 `None` 值。此时，Rust 将在编译期报错，提示我们在 irrefutable 模式下，哪些模式没有得到匹配：

```
error[E0005]: refutable pattern in local binding: `None` not covered
 --> <anon>:3:5
  |
3 | let Some(x) = some_option_value;
  |     ^^^^^^^ pattern `None` not covered
```

解决的方法就是我们使用 refutable 的 `if let`：

```rust
if let Some(x) = some_option_value {
    println!("{}", x);
}
```

而如果我们给 `if let` 一个 irrefutable 的模式，Rust 也不会通过编译：

```rust
if let Some(x) = some_option_value {
    println!("{}", x);
}
```

报错内容为：

```
error[E0162]: irrefutable if-let pattern
 --> <anon>:2:8
  |
2 | if let x = 5 {
  |        ^ irrefutable pattern
```

## 所有的模式匹配语法

### 字面量匹配

```rust
let x = 1;

match x {
    1 => println!("one"),
    2 => println!("two"),
    3 => println!("three"),
    _ => println!("anything"),
}
```

`_` 反映了剩余所有的状况。

### 具名变量的匹配

```rust
fn main() {
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(y) => println!("Matched, y = {:?}", y),
        _ => println!("Default case, x = {:?}", x),
    }

    println!("at the end: x = {:?}, y = {:?}", x, y);
}
```

注意，`match` 中的模式匹配子句会发生**变量遮蔽**。例如，上例中我们会命中第二条子句，该子句中的 `y` 不是外层声明的 `y` 。这个 `y` 将绑定到匹配到该子句的 `Some` 中的值，这里即是 `x`，值是 `5`。

### 多重匹配

```rust
let x = 1;

match x {
    1 | 2 => println!("one or two"),
    3 => println!("three"),
    _ => println!("anything"),
}
```

### 区间匹配

```rust
let x = 5;

match x {
    1 ... 5 => println!("one through five"),
    _ => println!("something else"),
}
```

```rust
let x = 'c';

match x {
    'a' ... 'j' => println!("early ASCII letter"),
    'k' ... 'z' => println!("late ASCII letter"),
    _ => println!("something else"),
}
```

### 解构匹配

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x: a, y: b } = p;
    assert_eq!(0, a);
    assert_eq!(7, b);
}
```

```rust
fn main() {
    let p = Point { x: 0, y: 7 };

    match p {
        Point { x, y: 0 } => println!("On the x axis at {}", x),
        Point { x: 0, y } => println!("On the y axis at {}", y),
        Point { x, y } => println!("On neither axis: ({}, {})", x, y),
    }
}
```

第一个匹配子句可以理解为，匹配到  `y` 为 `0`，`x` 任意的 `Point`。

### 解构枚举

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let msg = Message::ChangeColor(0, 160, 255);

    match msg {
        Message::Quit => {
            println!("The Quit variant has no data to destructure.")
        },
        Message::Move { x, y } => {
            println!(
                "Move in the x direction {} and in the y direction {}",
                x,
                y
            );
        }
        Message::Write(text) => println!("Text message: {}", text),
        Message::ChangeColor(r, g, b) => {
            println!(
                "Change the color to red {}, green {}, and blue {}",
                r,
                g,
                b
            )
        }
    }
}
```

### 解构引用

如果我们匹配的值包含了一个引用，我们需要解引用，此时可以在模式中声明 `&` 。这让我们获得了引用指向的值而不是引用。下例中，`&Point{x, y}` 就能让我们获得引用指向的 `Point`，并从中提取出 `x` 和 `y`。

```rust
let points = vec![
    Point { x: 0, y: 0 },
    Point { x: 1, y: 5 },
    Point { x: 10, y: -3 },
];

let sum_of_squares: i32 = points
    .iter()
    .map(|&Point { x, y }| x * x + y * y)
    .sum();
```

### 嵌套解构

```rust
let ((feet, inches), Point {x, y}) = ((3, 10), Point { x: 3, y: -10 });
```

### ignore

```rust
fn foo(_: i32, y: i32) {
    println!("This code only uses the y parameter: {}", y);
}

fn main() {
    foo(3, 4);
}
```

`_` 将忽略传入的第一个参数 `3`。

```rust
let mut setting_value = Some(5);
let new_setting_value = Some(10);

match (setting_value, new_setting_value) {
    (Some(_), Some(_)) => {
        println!("Can't overwrite an existing customized value");
    }
    _ => {
        setting_value = new_setting_value;
    }
}

println!("setting is {:?}", setting_value);
```

```rust
let numbers = (2, 4, 8, 16, 32);

match numbers {
    (first, _, third, _, fifth) => {
        println!("Some numbers: {}, {}, {}", first, third, fifth)
    },
}
```

变量名前加上 `_` ，可以让 Rust 在编译器跳过该变量是否使用的检查：

```rust
fn main() {
    let _x = 5;
    let y = 10;
}
```

这段代码在编译时，只会提示 `y` 没有被使用。

这个特性适合的场景为：我们需要变量赋值，又不想使用赋值后变量：

```rust
let s = Some(String::from("Hello!"));

if let Some(_s) = s {
    println!("found a string");
}

println!("{:?}", s);
```

### 使用 `..` 忽略剩余参数

```rust
struct Point {
    x: i32,
    y: i32,
    z: i32,
}

let origin = Point { x: 0, y: 0, z: 0 };

match origin {
    Point { x, .. } => println!("x is {}", x),
}
```

```rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (first, .., last) => {
            println!("Some numbers: {}, {}", first, last);
        },
    }
}
```

但是，如果 `..` 会带来**歧义**的话，Rust 是不会编译通过的：

```rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (.., second, ..) => {
            println!("Some numbers: {}", second)
        },
    }
}
```

在这个代码中，Rust 不知道这个 `second` 是第一匹配到的，还是某次匹配到的：

```
error: `..` can only be used once per tuple or tuple struct pattern
 --> src/main.rs:5:22
  |
5 |         (.., second, ..) => {
  |                      ^^
```

### `ref` 以及 `ref mut`

```rust
let robot_name = Some(String::from("Bors"));

match robot_name {
    Some(name) => println!("Found a name: {}", name),
    None => (),
}

println!("robot_name is: {:?}", robot_name);
```

这段代码无法通过编译，因为 `robot_name` 已经 move 到了 `match` 的匹配子句中了。

通过 `ref` 或者 `ref mut` 关键字，我们可以只是引用匹配对象，之所以不使用引用符号 `&`，是为了和解构引用进行区分。在解构引用中，\`&` 不会创建引用，只是进行匹配，看参数是否匹配到引用。

```rust
let robot_name = Some(String::from("Bors"));

match robot_name {
    Some(ref name) => println!("Found a name: {}", name),
    None => (),
}

println!("robot_name is: {:?}", robot_name);

```

```rust
let mut robot_name = Some(String::from("Bors"));

match robot_name {
    Some(ref mut name) => *name = String::from("Another name"),
    None => (),
}

println!("robot_name is: {:?}", robot_name);
```

### 匹配防御

通过在匹配条件中使用 `if`，我们可以更加精确地定义匹配条件：

```rust
fn main() {
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(n) if n == y => println!("Matched, n = {:?}", n),
        _ => println!("Default case, x = {:?}", x),
    }

    println!("at the end: x = {:?}, y = {:?}", x, y);
}
```

```rust
let x = 4;
let y = false;

match x {
    4 | 5 | 6 if y => println!("yes"),
    _ => println!("no"),
}
```

### `@` 绑定

```rust
enum Message {
    Hello { id: i32 },
}

let msg = Message::Hello { id: 5 };

match msg {
    Message::Hello { id: id_variable @ 3...7 } => {
        println!("Found an id in range: {}", id_variable)
    },
    Message::Hello { id: 10...12 } => {
        println!("Found an id in another range")
    },
    Message::Hello { id } => {
        println!("Found some other id: {}", id)
    },
}
```

使用 `@` ，可以将匹配到的值绑定到 `@` 声明的变量。