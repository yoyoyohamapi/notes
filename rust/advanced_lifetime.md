# Advanced Lifetime

这一章节中，将覆盖尚未提到的 Lifetime 特性：

- **Lifetime subtyping**：保证某个生命期比另一生命期存活更久。
- **Lifetime bounds**：为泛型绑定生命期。
- **Inference of trait object lifetimes** ：Trait 对象的生命期推断。

## Lifetime Subtyping

假定我们正在撰写一个 parser，并且有一个 `Context` 结构体来持有待 parse 的字符串。我们的 parser 将解析该字符串并返回成功与否。parser 需要借用 context 完成 parse：

```rust
struct Context(&str); // Context 是一个 tuple struct

struct Parser {
    context: &Context,
}

impl Parser {
    fn parse(&self) -> Result<(), &str> {
        Err(&self.context.0[1..])
    }
}
```

为了简化例子，`parse` 方法不会在成功时做任何事，在解析失败时，将返回解析失败的字符串分片。

由于我们没有声明生命期，编译这段代码将报错。因此，我们做出如下修改：

```rust
struct Context<'a>(&'a str);

struct Parser<'a> {
    context: &'a Context<'a>,
}

impl<'a> Parser<'a> {
    fn parse(&self) -> Result<(), &str> {
        Err(&self.context.0[1..])
    }
}
```

可以看到，Parser 和 Context 的参数都具有相同的生命期。我们告诉 Rust，`Parser`  将持有一个生命期为 `'a`  的 `Context` 引用，而 `Context` 又将持有同样生命期的字符串。

接下来，我们将撰写一个函数对 context 进行解析：

```rust
fn parse_context(context: Context) -> Result<(), &str> {
    Parser { context: &context }.parse()
}
```

编译代码，将报错：

```
error[E0597]: borrowed value does not live long enough
  --> src/lib.rs:14:5
   |
14 |     Parser { context: &context }.parse()
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ does not live long enough
15 | }
   | - temporary value only lives until here
   |
note: borrowed value must be valid for the anonymous lifetime #1 defined on the function body at 13:1...
  --> src/lib.rs:13:1
   |
13 | / fn parse_context(context: Context) -> Result<(), &str> {
14 | |     Parser { context: &context }.parse()
15 | | }
   | |_^

error[E0597]: `context` does not live long enough
  --> src/lib.rs:14:24
   |
14 |     Parser { context: &context }.parse()
   |                        ^^^^^^^ does not live long enough
15 | }
   | - borrowed value only lives until here
   |
note: borrowed value must be valid for the anonymous lifetime #1 defined on the function body at 13:1...
  --> src/lib.rs:13:1
   |
13 | / fn parse_context(context: Context) -> Result<(), &str> {
14 | |     Parser { context: &context }.parse()
15 | | }
   | |_^
```

错误内容可以概括为，`Parser` 实例和 `context` 参数都只能存活于**`Parser` 创建后到 `parse_context` 函数末尾**。因此，我们本寄希望于 `parse_content` 能返回 `Context` 持有的字符串分片，但它们已经无法存货了。

因此，`Parser` 和 `context` 都要能在**函数外部存活**，函数之前，之后都需要有效，以保证所有代码中的引用都保持有效。由于 `parse_context` 拿走了 `context` 的所有权，`Parse` 和 `context` 参数在函数末尾离开作用域时，将会 “死亡”。

回顾到之前 lifetime 省略原则，下面这段代码：

```rust
fn parse(&self) -> Result<(), &str> {}
```

将会被 Rust 自动声明生命期：

```rust
fn parse<'a>(&'a self) -> Result<(), &'a str> {}
```

可以看到，返回的字符串的生命期与 `Parse` 的生命期一致，即二者存活同样久，当 `Parse` 死亡时，`str` 也死亡。因此，当 `Parse` 实例在 `parse_context` 末尾离开作用域而死亡时，`parse()` 返回的字符串分片也将死亡，外部根本拿不到它。

现在，我们尝试为 `Parse` 和 `Context` 和声明不同的生命期：

```rust
struct Context<'s> (&'s str);

struct Parser<'c, 's> {
    context: &'c Context<'s>
}

impl<'c, 's> Parser<'c, 's> {
    fn parse(&self) -> Result<(), &'s str> {
        Err(&self.context.0[1..])
    }
}

fn parse_context(context: Context) -> Result<(), &str> {
    let parser = Parser { 
        context: &context
    };
    parser.parse()
}
```

然而还是报错：

```
error[E0491]: in type `&'c Context<'s>`, reference has a longer lifetime than the data it references
 --> src/lib.rs:4:5
  |
4 |     context: &'c Context<'s>,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^
  |
note: the pointer is valid for the lifetime 'c as defined on the struct at 3:1
 --> src/lib.rs:3:1
  |
3 | / struct Parser<'c, 's> {
4 | |     context: &'c Context<'s>,
5 | | }
  | |_^
note: but the referenced data is only valid for the lifetime 's as defined on the struct at 3:1
 --> src/lib.rs:3:1
  |
3 | / struct Parser<'c, 's> {
4 | |     context: &'c Context<'s>,
5 | | }
  | |_^
```

Rust 并不知道两个生命期 `'c` 和 `'s` 的关系。我们知道， `'s` 需要比 `'c` 存活更久，这样 `parse_content` 才能返回有效地引用。

此时，借助于 Lifetime Subtyping，通过 `'b:'a` 这样的语法，我们能够声明 `'b` 至少与 `'a` 存活得一样久。

```rust
struct Parser<'c, 's: 'c> {
    context: &'c Context<'s>,
}
```

注意，这里不是指 `'s` 和 `'c` 生命期一样，这样做就没有意义了。这里只是保证了当 `Context` 走向死亡时，`Context` 持有的字符串引用不会走向死亡。因此，`parse_content` 中，当 `context` 参数离开作用域时，其持有的字符串引用不会消亡。

## Lifetime bounds

在 smart pointer 一章中，我们知道，`RefCell<T>` 的 `borrow` 和 `borrow_mut` 分别会返回 `Ref` 和 `RefMut` 类型，这些类型会对引用做一个 wrapper，以在运行时跟踪借用规则。 `Ref` 的结构体可能为：

```rust
struct Ref<'a, T>(&'a T);
```

尝试编译该代码，将会报错：

```
error[E0309]: the parameter type `T` may not live long enough
 --> src/lib.rs:1:19
  |
1 | struct Ref<'a, T>(&'a T);
  |                   ^^^^^^
  |
  = help: consider adding an explicit lifetime bound `T: 'a`...
note: ...so that the reference type `&'a T` does not outlive the data it points at
 --> src/lib.rs:1:19
  |
1 | struct Ref<'a, T>(&'a T);
  |                   ^^^^^^
```

 `T` 作为一个泛型，可以是任意类型，因此，如果 `T` 是一个引用，或者包含了引用，Rust 无法根据现有信息保证到引用有效，因此，我们需要为泛型绑定生命期：

```rust
struct Ref<'a, T: 'a>(&'a T);
```

通过这样的声明，Rust 现在知道，如果 `T` 包含了任意引用，那么引用存活时间不能比 `'a` 少，亦即，不会再结构体 `Ref` 消亡后消亡。

我们也可以用另一个方式解决该问题：

```rust
struct StaticRef<T: 'static>(&'static T);
```

这样，`T` 会在整个程序中都保持有效。

## Inference of trait object lifetimes

```rust
trait Red {}

struct Ball<'a> {
    diameter: &'a i32,
}

impl<'a> Red for Ball<'a> { }

fn main() {
    let num = 5;

    let obj = Box::new(Ball { diameter: &num }) as Box<Red>;
}
```

> `Box<Red>` 是一个 trait object，代表了 `Box` 中的对象实现了 `Red` trait。

上面这段代码中，即便我们没有任何显示地声明 `obj` 的生命期。这是通过下面这些规则完成的：

- trait object 默认的生命期为 `'static`
- 使用了 `&'a Trait` 或者 `&'a mut Trait`，则默认的生命期是 `'a`
- 使用了单个 `T: 'a` 从句，则生命期是 `'a`。
- 使用了多个类似 `T: 'a` 这样的从句的话，没有默认的生命期，需要我们显式声明。

上例中，如果我们要显示为 trait object 声明生命期的话，需要：`Box<Red + 'a>` 或者是 `Box<Red + 'static>`。这意味着任何 `Red` trait 的实现，且当中包含了引用，则 trait object 都需要绑定与引用相同的生命期。