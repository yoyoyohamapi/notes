# Trait（特性）

Trait 类似接口，用来承载**共享行为**，首先，使用 trait 来声明一个特性。例如，下面我们声明了 一个 `Summarizable` trait，要让某个对象是 Summarizable 的，就要实现 `summary` 方法

```rust
pub trait Summarizable {
    fn summary(&self) -> String;
}
```

例如，一篇新闻就是 Summrizable 的，其 summary 就是时间、地点、人物；一篇推文也是 Summrizable 的，它的 summary 就是用户和内容。

```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summarizable for NewsArticle {
    fn summary(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summarizable for Tweet {
    fn summary(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

比如，我们的 `NewsArticle` 和 `Tweet` 都实现了`Summarizable` 特性，接下来可以这么使用：

```rust
let tweet = Tweet {
    username: String::from("horse_ebooks"),
    content: String::from("of course, as you probably already know, people"),
    reply: false,
    retweet: false,
};

println!("1 new tweet: {}", tweet.summary());
```

也可以为 Trait 的方法指定**默认**的行为：

```rust
pub trait Summarizable {
    fn summary(&self) -> String {
        String::from("(Read more...)")
    }
}

impl Summarizable for NewsArticle {}

let article = NewsArticle {
    headline: String::from("Penguins win the Stanley Cup Championship!"),
    location: String::from("Pittsburgh, PA, USA"),
    author: String::from("Iceburgh"),
    content: String::from("The Pittsburgh Penguins once again are the best
    hockey team in the NHL."),
};

println!("New article available! {}", article.summary());
```

## Trait Bounds

可以使用 Trait 精细化定义泛型：

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: T, u: U) -> i32 {}
```

这个定义也可以通过 `where` 进行改在：

```rust
fn some_function<T, U>(t: T, u: U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{}
```

回顾泛型一节的例子：

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

该例子将报错：

```
error[E0369]: binary operation `>` cannot be applied to type `T`
  |
5 |         if item > largest {
  |            ^^^^
  |
note: an implementation of `std::cmp::PartialOrd` might be missing for `T`
```

现在，改造这个例子，代码将能够通过编译：

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}

```

这是因为比较操作符需要对象具备 `PartialOrd` trait，而对象赋值则需要具备 `Copy` trait。

## 使用 Trait Bounds 有条件地实现某方法

下面，我们定义了一个简单的结构体，并且为其实现了 `new` 方法：

```rust
use std::fmt::Display;

struct Pair<T> {
  x: T,
  y: T,
}

impl<T> Pair<T> {
  fn new(x: T, y: T) -> Self {
    Self {x, y}
  }
}
```

如果泛型 T 实现了 `Display` 和 `PartialOrd` Trait，我们就为这个 Pair 实现 `cmp_display()` 方法：

```rust
impl<T: Display + PartialOrd> Pair<T> {
  fn cmp_display(&self) {
    if self.x >= self.y {
      println!("The largest member is x = {}", self.x);
    } else {
      println!("The largest member is y = {}", self.y);
    }
  }
}
```

我们也可以为实现了某个 Trait 的泛型继续实现另外的 Trait：

```rust
impl<T: Display> ToString fro T {
  // ---snip---
}
```

