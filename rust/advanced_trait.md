# Advanced Trait

## 关联类型占位符

trait 中可以通过 `type` 指定 trait 的关联类型（Associated Types），后续 trait 方法中都可以使用占位符制定的类型。例如在 `Iterator ` 中，我们通过类型占位符，制定了迭代器中对象类型： 

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        // --snip--
```

相较于泛型，占位符更应该被考虑为 trait 中各个方法都可能关联的类型，trait 所依托的类型。上面的需求中，如果我们用泛型替换之，则每个用到该泛型的方法，我们都需要逐一声明泛型：

```rust
pub trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```

## 默认泛型参数及运算符重载

Rust 不允许自定义运算符，但是允许用户对运算符进行重载，运算符 trait 都位于 `std::ops` 中。下面的代码中，自定义的 `Point` 重载了加法运算符：

```rust
use std::ops::Add;

#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32
}

impl Add for Point {
    type Output = Point;
    
    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y
        }
    }
}

fn main() {
    assert_eq!(Point { x: 1, y: 0} + Point { x: 2, y: 3}),
    		   Point { x: 3, y: 3});
}
```

我们看下 `Add`  trait 的定义：

```rust
trait Add<RHS=Self> {
    type Output;
    
    fn add(self, rhs: RHS) -> Self::Output;
}
```

通过 `<RHS=Self>` ，我们为 `Add` trait 声明了 rhs （right hand side）的默认泛型类型为当前类型。这样，当我们实现 `Add` trait 时，如果我们不显示声明具体类型，则参数 `rhs` 的类型为 `Self`。

而下面的代码中，我们则显式声明了 `rhs` 的默认类型：

```rust
use std::ops::Add;

struct Millimeters(u32);
struct Meters(u32);

impl Add<Meters> for Millimeters {
    type Output = Millimeters;
    
    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + (other.0 * 1000))
    }
}
```

现在，加法右侧的数据类型为 `Meters`，而加法的返回结果则为 `Millimeters`。

## 使用 **fully qualified syntax** 进行方法调用

看到下面的代码，`Human` 实例下有多个同名的 `fly` 方法：

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}

impl Human {
    fn fly(&self) {
        println!("*waving arms furiously*");
    }
}
```

尝试直接在实例上调用 `fly` 方法：

```rust
fn main() {
    let person = Human;
    person.fly();
}
```

程序将调用 `person` 自己的 `fly` 方法，输出：`*waving arms furiously*`。如果我们想调用所实现的 trait 上的方法：

```rust
fn main() {
    let person = Human;
    Pilot::fly(&person);
    Wizard::fly(&person);
    person.fly();
}
```

之所以这样调用，是因为 `Pilot` 等 trait 的 `fly` 方法的第一个参数都是 `&self`。

然而看到下面的一个例子，倘若我们实现的 trait 方法没有 `&self` 参数，即此时 `baby_name` 是一个关联函数（associated function），而不是一个方法：

```rust
trait Animal {
    fn baby_name() -> String;
}

struct Dog;

impl Dog {
    fn baby_name() -> String {
        String::from("Spot")
    }
}

impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}

fn main() {
    println!("A baby dog is called a {}", Dog::baby_name());
}
```

我们本预期该代码能够输出：

```
A baby dog is called a puppy
```

但它却输出了：

```
A baby dog is called a Spot
```

然后我们替换 `main` 中的方法调用为：

```rust
fn main() {
    println!("A baby dog is called a {}", Animal::baby_name());
}
```

由于 `Animal::baby_name` 是一个关联函数而不是方法，Rust 不知道要调用该方法的哪个实现，因此，将在编译期报错：

```
error[E0283]: type annotations required: cannot resolve `_: Animal`
  --> src/main.rs:20:43
   |
20 |     println!("A baby dog is called a {}", Animal::baby_name());
   |                                           ^^^^^^^^^^^^^^^^^
   |
   = note: required by `Animal::baby_name`
```

此时，我们可以使用 **fully qualified syntax** 来指明要调用哪个实现：

```rust
fn main() {
    println!("A baby dog is called a {}", <Dog as Animal>::baby_name());
}
```

**fully qualified syntax** 可以被描述为：

```
<Type as Trait>::function(receiver_if_method, next_arg, ...);
```

## Supertrait

有时候，我们需要在某个 trait 中，使用到其他 trait 的特性，例如，我们有一个 `OutlintPrint` trait，它有一个 `outline_print` 方法，该方法能够在值得周围打印一圈 `*` 号。给定一个实现了 `Display` trait 的 `Point` 结构体，能输出如下：

```
**********
*        *
* (1, 3) *
*        *
**********
```

为了在 `outline_print` 中使用到 `Display` trait 的特性，我们需要声明 `OutlinePrint`  trait 只有在类型实现了 `Display` 的情况下才能工作：

```rust
use std::fmt;

trait OutlinePrint: fmt::Display {
    fn outline_print(&self) {
        let output = self.to_string();
        let len = output.len();
        println!("{}", "*".repeat(len + 4));
        println!("*{}*", " ".repeat(len + 2));
        println!("* {} *", output);
        println!("*{}*", " ".repeat(len + 2));
        println!("{}", "*".repeat(len + 4));
    }
}
```

因为我们已经声明了 `OutlinePrint` trait 需要 `Display` trait，因此可以再起方法中使用类型自己实现的 `to_string` 方法。测试代码如下：

```rust
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

impl OutlinePrint for Point {}
```

## Newtype：在外部类型上实现外部 Trait

假定我们想要在 `Vec` 上实现 `Display` 方法，orphan rules 将不允许我们这么做。

> orphan rule：只有 trait 和 type 其中之一相对于 crate 是 local 的情况下，我们才能为 type实现 trait。

而 `Display` 和 `Vec` 都定义在了外部，因此我们需要借助一个 `Wrapper` 让 type 落到 local 下，间接地为 type 实现 trait：

```rust
use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("hello"), String::from("world")]);
    println!("w = {}", w);
}
```

在这段代码中，我们通过 `self.0` 取到了 `Wrapper` 包裹的 `Vec`，并为其实现了格式换输出的方法。

`Wrapper` 扮演了一个 newtype 的角色，并不会带来运行时的开销，在编译器，Rust 会去除包裹。

这个做法的一个缺陷是，`Wrapper` 作为一个 new type，不具有 `Vec` 自身任何的方法。如果我们想要 new type 拥有 inner type 每个方法，可以实现 `Deref` trait 来获得 innert type。如果我们 wrapper 拥有 inner type 的所有方法，我们只需要实现我们想要的方法即可。