# Collection（集合）

## Vector

### 无效的引用

```rust
let mut v = vec![1, 2, 3, 4, 5]

let first = &v[0];

v.push(6);
```

编译这段代码，将会报错：

```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 -->
  |
4 |     let first = &v[0];
  |                  - immutable borrow occurs here
5 |
6 |     v.push(6);
  |     ^ mutable borrow occurs here
7 |
8 | }
  | - immutable borrow ends here
```

这段代码报错的原因与 vector 的工作机制有关：添加一个新的元素到 vector 尾部需要进行内存分配，如果无法在原来的空间继续申请一个连续的内存保存新的元素，那么就需要一段新的内存区域，并且将老的元素都拷贝到新的内存区域。因此，根据借用原则，在拥有了不可变引用后，不能再拥有可变引用。

### 使用枚举来存储多个类型

```rust
enum SpreadsheetCell {
    Int(i32),
    Float(f64),
    Text(String),
}

let row = vec![
    SpreadsheetCell::Int(3),
    SpreadsheetCell::Text(String::from("blue")),
    SpreadsheetCell::Float(10.12),
];
```

## Strings

一个 `String` 可以看做是 `Vec<u8>` 的包裹。

```rust
let len = String::from("Hola").len();
```

这里的长度返回的是 4，因为各个字符都能使用了一个字节进行 UTF8 编码。但是下面这段代码：

```rust
let len = String::from("Здравствуйте").len()
```

你可能会觉得这里返回的长度是 12，但实际上是 24。因为这段字符串中的各个字符都需要 2 个字节进行 UTF8 编码。

那么下面这段代码同样无法返回预期的 `З`，而是会返回第一个字节 `208`：

```rust
let hello = "Здравствуйте";
let answer = &hello[0];
```

所以，Rust 就允许这样的代码编译通过，从源头遏制住对字符串取值可能造成的误会或者错误。

### 连接字符串

```rust
let s1 = String::from("Hello, ");
let s2 = String::from("world!");
let s3 = s1 + &s2; 
```

`s1` 的所有权将被转移，之后其不再可用。这是因为 `+` 的实现可以看做是：

```rust
fn add(self, s: &str) -> String { }
```

通过函数签名可以看到，`self` 会被拿走所有权.

所以，更建议使用 `format!` 宏来连接字符串，它不会拿走任何参数的所有权：

```rust
let s1 = String::from("tic");
let s2 = String::from("tac");
let s3 = String::from("toe");

let s = format!("{}-{}-{}", s1, s2, s3);
```

### `नमस्ते`

`नमस्ते` 这个字符串最终会以 `u8` 类型的 vector 进行存储：

```
[224, 164, 168, 224, 164, 174, 224, 164, 184, 224, 165, 141, 224, 164, 164,
224, 165, 135]
```

进行迭代的话：

```rust
for b in "नमस्ते".bytes() {
    println!("{}", b);
}
```

将会输出：

```
224
164
168
224
// ... etc
```



如果我们以 Unicode scalar 的方式看，即 Rust 中 `char` 类型：

```
['न', 'म', 'स', '्', 'त', 'े']
```

其中第四和第六个字符为变音符号。进行迭代的话：

```rust
for c in "नमस्ते".chars() {
    println!("{}", c);
}
```

将输出：

```
न
म
स
्
त
े
```

最后一个 Rust 不允许我们队字符串进行的索引的原因是，通常，为了获得一个字符串中的某个字符，将会进行时间复杂度为 `O(1)` 的操作，但是这对于字符串来说，较难保证，Rust 必须从字符串开始进行字符串内容的遍历，才能取到有效字符。

如果真的需要取某个字符，使用 slice 来精确描述你要取值的字节范围：

```rust
let hello = "Здравствуйте";

let s = &hello[0..4];
```

这里，`s` 是一个 `&str`，其指向的内容为 `Зд`。

而如果你尝试使用 `&hello[0..1]` 呢 ？ Rust 将在运行时 panic：

```
thread 'main' panicked at 'byte index 1 is not a char boundary; it is inside 'З' (bytes 0..2) of `Здравствуйте`', src/libcore/str/mod.rs:2188:4
```

## HashMap

```rust
use std::collections::HashMap;

let field_name = String::from("Favorite color");
let field_value = String::from("Blue");

let mut map = HashMap::new();
map.insert(field_name, field_value);
```

之后，`field_name`、`field_value` 将不能再使用，因为 `insert()` 方将会拿走二者的所有权。如果使用二者的引用，那么值不会被送入 hash map，因为 Rust 要保证引用的值存活的时间不低于 hash map 存活的时间。

