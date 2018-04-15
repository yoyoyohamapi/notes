# 在 Rust 进行面向对象编程

  我们将在 Ruts 中通过一个博客小程序探索 Rust 中如何进行 OO 编程，这个小程序将完成如下功能：

1. 新建的博文为一个空的草稿。
2. 如果撰写完成，就需要对博文进行 review。
3. 如果 review 通过，则博文可以进行发布。
4. 只有发布的博文内容会被输出。

最终的测试如下：

```rust
extern crate blog;
use blog::Post;

fn main() {
    let mut post = Post::new();
    
    post.add_text("I ate a salad for lunch today.")
    assert_eq!("", post.content());
    
    post.request_review();
    assert_eq!("", post.content());
    
    post.approve();
    assert_eq!("I ate a salad for lunch today", post.content());
}
```

## 定义 `Post` 类

src/lib.rs：

```rust
pub struct Post {
    state: Option<Box<State>>,
    content: String
}

impl Post {
    pub fn new() -> Post {
        Post {
            state: Some(),
            content: String::new(),
        }
    }
}

impl Post {
    // 构造方法
    pub fn new() -> Post {
        // 新建的博文状态为 `Draft`
        Post {
            state: Some(Box::new(Draft {})),
            content: String::new()
        }
    }
    
    // 添加内容
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
    
    // 获得内容
    pub fn content(&self) -> &str {
        ""
    }
}

trait State {}

struct Draft {}

impl State for Draft {}
```

我们定义了 `Post` 类，它有两个属性：

- `state`：博文状态。这里使用 `Box<T>` 是因为，各个不同 state struct 大小不一，需要在运行时进行堆上的内存分配。
- `content`：博文内容

以及三个方法：

* `new`：构造方法，用来实例化 Post 对象
* `add_text`：添加博文内容
* `content`：获得博文内容

定义了 `State` trait，它定义了博文的状态，`Draft`、`PendingReview`、`Published` 等状态都将实现这个 `trait`。

## 请求审核

接下来，将为博文类实现一个 `request_review` 方法，该方法将博文的状态由 `Draft` 改为 `PendingReview`：

```rust
impl Post {
    // --snip--
    pub fn request_review(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.request_review())
        }
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<State>;
}

struct Draft {}

impl State for Draft {
    fn request_review(self: Box<Self>) -> Box<State> {
        Box::new(PendingReview{})
    }
}

struct PendingReview {}

impl State for PendingReview {
    fn request_review(self: Box<Self>) -> Box<State> {
        self
    }
}
```

- `State` trait 定义了 `request_review` 方法，这样，实现了 `State` trait 的状态结构就能定义各自在博文进行 `request_review` 时的行为。例如，一个状态为 `Draft` 的博文要求 review 时，其状态就会变更为 `PendingReview`。
- `request_review` 的函数签名是 `self: Box<Self>`，而不是 `self`、`&self` 或者 `&mut self`。这意味着方法调用时，`Box<Self>` 会被拿走所有权，因此旧的 `state` 将不再有效，`Post` 对象将转换其 `state` 为新的 `state`。
- 为了消费掉旧的 `state`，`request_view` 方法就需要拿走其所有权。这也是为什么使用了 `Option`，`take` 方法会从 `state` field 中提取出 `Some`，并在 field 留下一个 `None`，这让我们将 `state` 从 `Post` 中提取出来，而不是借用它。通过 `if` 判断，我们保证当前从 `state` field 消耗了旧的 state，并返回新的 `state`。

## 通过审核

类似地，审核通过的方法如下，我们新声明了 `approve` 方法和 `Publish` 状态：

```rust
impl Post {
    // --snip--
    pub fn approve(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.approve())
        }
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<State>;
    fn approve(self: Box<Self>) -> Box<State>;
}

struct Draft {}

impl State for Draft {
    // --snip--
    fn approve(self: Box<Self>) -> Box<State> {
        self
    }
}

struct PendingReview {}

impl State for PendingReview {
    // --snip--
    fn approve(self: Box<Self>) -> Box<State> {
        Box::new(Published {})
    }
}

struct Published {}

impl State for Published {
    fn request_review(self: Box<Self>) -> Box<State> {
        self
    }

    fn approve(self: Box<Self>) -> Box<State> {
        self
    }
}
```

## 根据状态获得内容

我们根据当前博文的状态，来获得博文内容，只有在博文发布了，才能获取其内容，否则返回空。可以这么做：

```rust
impl Post {
    // --snip--
    pub fn content(&self) -> &str {
        self.state.as_ref().unwrap().content(&self)
    }
    // --snip--
}

trait State {
    // --snip--
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        ""
    }
}

// --snip--
struct Published {}

impl State for Published {
    // --snip--
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        &post.content
    }
}
```

- 我们使用了`Option` 上的  `as_ref` 方法，因为我们只想获得 `Option` 内部的内容，而不向拿走其所有权。
- 由于 `state` 是 `Option<Box<State>>` 类型，调用 `as_ref` 将返回 `Option<&Box<State>>`。
- 由于 `state` 不会是一个 `none`，因此我们放心地通过 `unwrap` 取出其 `Some` 值，即 `&Box<State>`。
- 根据 Derek Coercion，`&Box<State>` 将自动完成解引用，从而调用到 `State` 上的 `content` 方法。

## 另一个方案

在上述的方案中，`Post` 是通过其 `state` 属性进行状态转移的，我们换一种角度思考，可以定义不同 Post 类型，并实现它们之间的转移：

```rust
pub struct Post {
    content: String,
}

pub struct DraftPost {
    content: String,
}

impl Post {
    pub fn new() -> DraftPost {
        DraftPost {
            content: String::new(),
        }
    }

    pub fn content(&self) -> &str {
       &self.content
    }
}

impl DraftPost {
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
}
```

现在，直接在 `request_review` 中返回状态进行类型转换：

```rust
impl DraftPost {
    // --snip--

    pub fn request_review(self) -> PendingReviewPost {
        PendingReviewPost {
            content: self.content,
        }
    }
}

pub struct PendingReviewPost {
    content: String,
}

impl PendingReviewPost {
    pub fn approve(self) -> Post {
        Post {
            content: self.content,
        }
    }
}
```

## 总结

Rust 中能够以 OO 进行开发，但是 OO 不一定是最佳选择。