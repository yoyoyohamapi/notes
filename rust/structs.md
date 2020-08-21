# Struct（结构体）

## 定义一个结构体

```rust
struct User {
  username: String;
  email: String;
  sign_in_count: u64;
  active: bool,
}
```

## 实例化一个结构体

```rust
let user1 = User {
  email: String::from("someone@example.com"),
  username: String::from("someusername123"),
  active: true,
  sign_in_count: 1,
};
```

## 工厂函数

```rust
fn build_user(email: String, username: String) -> User {
  User {
    email,
    username,
    active: true,
    sign_in_count: 1,
  }
}
```

## Tuple Struct

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

let black = Color(0, 0, 0);
let origin = Point(0, 0, 0);
```

## Demo

```rust
struct Rectangle {
  width: u32,
  height: u32,
}

fn main() {
  let rect1 = Rectangle {
    width: 30,
    height: 50,
  };
  
  println!(
    "The area of the rectangle is {} square pixels.",
    area(&rect1)
  );
}

fn area(rectangle: &Rectangle) -> u32 {
  rectangle.width * rectangle.height;
}
```

## Derived Trait

```rust
#[derive(Debug)]
struct Rectangle {
  width: u32,
  height: u32,
}

fn main() {
  let rect1 = Rectangle {
    width: 30,
    height: 50,
  };
  
  println!("retc1 is: {:?}", rect1);
}
```

