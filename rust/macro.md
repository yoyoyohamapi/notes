# Macro（宏）

Rust 中的 macro 分两类：

- 使用 `macro_rules` 定义的声明式宏（declarative macro）

- 三种过程式宏（procedural macro）：接受  Token Stream 作为输入，输出 Token Stream

  - Derive macro，通常用于修饰结构体和枚举

  - Attribute-like macro，更加自由，以属性的方式修饰任何对象

  - Function-like macro，通过 `!` 调用的宏

    

## 声明式宏

通过 `macro_rules!` 可以直接声明一个宏，类似于模式匹配，声明式可以为宏指定要匹配的 pattern，当 pattern 满足时，pattern 所对应的 block 中的代码就会被执行：

```rust
#[macro_export]
macro_rules! vec {
  ($( $x:expr ),* ) => {
    {
      let mut temp_vec = Vec::new();
      $(
        temp_vec.push($x);
      )*
      temp_vec
    }
  }
}
```

上述定义的 pattern 就是 `( $( $x:expr ),* )`，`$( $x:expr )` 俘获了要匹配的代码，方便在之后进行替换（这里就替换成了 `$(temp_bec.push($x);)*`）。

`$(x: expr)` 会命中所有传递给宏的表达式，并且将命中的表达式赋值给 `$x`，紧跟在 `$()` 之后的 `,` 则指出了 `,` 可以出现在命中条件的代码之后，`* ` 则指出了前面的 pattern 运行命中 0 次或者多次。

接下来，就可以使用这个宏创建 Vector 了：

```rust
let v: Vec<u32> = vec![1, 2, 3];
```

## 过程式宏

过程式的宏则不是通过声明 pattern 来创建宏的，而是通过一个接受代码作为输入，并输出代码的函数来创建的：

```rust
use proc_macro;

#[some_attribute]
pub fn some_name(input: TokenStream) -> TokenStream {}
```

### Derive macro

假定我们定义了一个 Trait：

```rust
pub trait HelloMacro {
  fn hello_macro();
}
```

我们要为一个 Struct 实现这个 Trait，就需要：

```rust
use hello_macro:HelloMacro;

struct Pancakes;

impl HelloMacro for Pancakes {
  fn hello_macro() {
    println!("Hello, Macro! My name is Pancakes!");
  }
}

fn main() {
  Pancakes::hello_macro();
}
```

借助于 `#[proc_macro_derive(Trait)]`来创建一个宏，这个宏能够为 Struct 修饰传入的 Trait，从而砍掉不少样板代码：

```rust
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;
use syn;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
  let ast = syn::parse(input).unwrap();
  
  impl_hello_macro(&ast)
}

fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident;
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };
    gen.into()
}
```

我们把样板代码内聚到了宏内部，现在，就可以这么为 `Pancakes` struct 实现 `HelloMacro` trait 了：

```rust
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}
```

### Attribute-like macro

Attribute-like macro 则更为自由，通过 `#[proc_macro_attribute]` 能创建修饰任何对象的宏：

```rust
#[route(GET, "/")]
fn index() {}

#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {}
```

### Function-like macro

Function-like macro 使用 `#[proc_macro]` 创建，并使用 `!` 进行调用（这一点类似于 `macro_rules` 所声明的宏）：

```rust
let sql = sql!(SELECT * FROM posts WHERE id=1);

#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {}
```

