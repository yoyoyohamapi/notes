# Unsafe Rust

unsafe Rust 让 Rust 略过**内存安全的**保障。允许不安全的代码的理由有两点：

1. Rust 的静态分析是**保守的**。在编译期，Rust 秉承的是 “宁可错杀一千，也不放过一个”，因此，一段正确的代码可能会被 Rust 拒绝在编译期。
2. 计算机硬件不是安全的。如果 Rust 限制了我们只能撰写安全的代码，那么一些 low level 的特性无法为我们所用。要知道，Rust 是可以用来写操作系统的。

## Unsafe 提供的能力

我们可以使用 `unsafe` 关键字来声明一个不安全的代码块，它为我们提供了 safe code 不具备的几个能力：

1. 解引用 raw pointer（裸指针）
2. 调用不安全的函数或者方法
3. 访问、修改静态变量（static variable）
4. 实现不安全的 trait

需要注意的如下几点：

- `unsafe` 不会屏蔽掉 Rust 的借用规则，如果你在 `unsafe` 代码块中使用了引用，它仍会被 Rust 所检查。Rust 提供的不安全特性仅仅是以上四点，这四点不会被编译器以内存安全为由所拒绝。换言之，在 `unsafe` 块中的代码，不总是不安全的。
- `unsafe` 块中的代码也并非就绝对是危险的，会造成内存问题的，只是 Rust 认为是潜在不安全的，需要程序员人为保证安全。`unsafe` 算是一个醒目的标语，提示我们在这里的代码，要严密注意内存安全。
- 尽可能的**隔离**不安全的代码，使用一个 safe 的抽象对其进行包裹，并暴露一个 safe 的 API。

## 解引用 raw pointer

unsafe Rust 中，存在一个类似引用的新类型 **raw pointer**，它也是可以是 immutable 或者 mutable 的，并使用 `*const T`、`*mut T` 进行声明。这里的 `*` 号不是解引用操作符，只是类型名的一个部分。在 raw pointer 的上下文中，“immutable” 指的是**指针被解引用后，不能再直接进行赋值**。

相较于引用和 smart pointer，raw pointer 一些特别的点是：

- 可以忽略借用规则，我们可以同时拥有 immutable 和 mutable 的 raw pointer，或者多个 mutable 的 raw pointer。
- 不保证指向有效的内存
- 允许为 null
- 没有实现任何的自动清理机制

通过这些特性，你可以获得更好的性能，也可以和其他语言以及更多的硬件特性交互，但是你可能舍去内存安全的保障。

```rust
let mut num = 5;

let r1 = &num as *const i32;
let r2 = &mut num as *mut i32;
```

上面这段代码中，通过 `as` 操作符，我们获得 `num` immutable 引用、mutable 引用对应的 raw pointer。这些 raw pointer 必须放在 `unsafe` 块中使用：

```rust
let mut num = 5;

let r1 = &num as *const i32;
let r2 = &mut num as *mut i32;

unsafe {
    println!("r1 is: {}", *r1);
    println!("r2 is: {}", *r2);
}
```

要注意的是，`*const i32` 和 `*mut i32` raw pointer 都指向了相同的内存位置，也就是 `num` 所在的位置。通过 raw pointer，我们同时拥有了相同内存位置的 immutable 和 mutable 的 pointer，这里可能造成一个数据竞态（data race）。

既然如此，为什么还要使用 raw pointer 呢？一个用例就是和 C 代码交互，另一个用例是构造一个借用规则无法理解的安全抽象。

## 调用不安全的函数或者方法

不安全的函数或者方法头部有一个额外的 `unsafe` 。`unsafe` 醒目地标识了这段函数需要额外注意，因为 Rust 无法保证安全。不安全的方法**必须**在 `unsafe` 块中进行调用：

```rust
unsafe fn dangerous() {}

unsafe {
    dangerous();
}
```

## 创建安全抽象

函数包含了 `unsafe` 的代码块不意味着整个函数都是不安全的。例如，Rust 中的 `split_at_mut`，该函数接收一个分片和中位数，根据中位数，将原分片一分为二，用法如下：

```rust
let mut v = vev![1, 2, 3, 4, 5, 6]
let r = &mut v[..];

let (a, b) = r.split_at_mut(3);

assert_eq!(a, &mut [1, 2, 3]);
assert_eq!(b, &mut [4, 5, 6]);
```

假定我们现在去实现该方法，可能会这么写：

```rust
fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len()
    
    assert!(mid <= len);
    
    (&mut slice[..mid], &mut slice[mid..])
}
```

很明显，我们同时拥有了多个 `slice` 的可变引用，违反了借用规则，因此无法通过编译：

```
error[E0499]: cannot borrow `*slice` as mutable more than once at a time
 -->
  |
6 |     (&mut slice[..mid],
  |           ----- first mutable borrow occurs here
7 |      &mut slice[mid..])
  |           ^^^^^ second mutable borrow occurs here
8 | }
  | - first borrow ends here
```

尽管我们的逻辑是正确的，但是 Rust 可不这么认为，因此，我们必须使用 `unsafe` 提供的能力了：

```rust
use std::slice;

fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut[i32], &mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();
    
    assert!(mid <= len);
    
    unsafe {
        (
            slice::from_raw_parts_mut(ptr, mid),
            slice::from_raw_parts_mut(ptr.offset(mid as isize), len - mid))
        )
    }
}
```

- `as_mut_ptr` 返回了 `slice` 的 raw pointer，类型是 `*mut i32`。
- `slice::from_raw_parts_mut` 函数接受了一个 raw pointer 和一个长度值，据此创建一个 slice。这个函数是不安全的，因为它必须新人 pointer 是有效的。
- `offset` 方法能够设置 raw pointer 的偏移量。该函数也是不安全的，因为它需要信任对应的偏移位置也是有效的。

我们不需要让 `split_at_mut` 函数是不安全的，这样我们可以在安全的 Rust 中调用该函数。事实上，因为我们保证了 raw pointer 位置的安全，因此确实可以认为我们这个函数是安全的。

作为对比，下面的 unsafe 的代码随意指定了内存位置，将造成意想不到的后果:

```rust
use std::slice;

let address = 0x012345usize;
let r = address as * mut i32;

let slice = unsafe {
    slice::from_raw_parts_mut(r, 10000)
}
```

## 使用 `extern` 来调用外部代码

Rust 是具备和其他语言交互的能力的。`extern` 带来了 FFI（Foreign Function Interface） 能力，FFI 能定义一些函数，让其他语言来调用这些函数。

下面的例子中，我们在 Rust 中使用了 C 语言的求绝对值函数：

```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```

`C` 定义了外部函数使用哪个 ABI（Application Binary Interface），ABI 定义了如何汇编层面调用函数。

### 在别的语言中调用 Rust 函数

我们也可以使用 `extern` 来创建一个供其他语言调用的接口。现在，我们不是声明一个 `extern` 块，而是在 `fn` 前声明 `extern`  关键字和所用的 ABI。我们也需要使用 `#[no_mangle]` 来告诉 Rust 不要对函数名进行混淆，这样才能让其他语言根据名称调用：

```rust
#[no_mangle]
pub extern "C" fn call_from_c() {
    println!("Just called a Rust function from C!");
}
```

## mutable static 变量的访问和修改

在 Rust 中，全局变量（global variable）被称为静态变量（static variable）：

```rust
static HELLO_WORLD: &str = "Hello, world!";

fn main() {

    println!("name is: {}", HELLO_WORLD);

}
```

静态变量**必须**声明类型。静态变量只会存储一个生命期是 `'static` 的引用，这意味我们无须为静态变量手动声明生命期。访问一个不可变的静态变量是安全的。

常量和不可变的静态变量可能看起来一样，但是静态变量由固定的内存地址，它总是访问相同的数据。常量则在他们被使用时，进行拷贝。

另一个常量和静态变量的区别是，静态变量可以是可变的。访问和修改可变的静态变量都是不安全的，我们需要使用 `unsafe` 块进行包裹：

```rust
static mut COUNTER: u32 = 0;

fn add_to_count(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    add_to_count(3);

    unsafe {
        println!("COUNTER: {}", COUNTER);
    }
}
```

## 实现 unsafe trait

```rust
unsafe trait Foo {
    // methods go here
}

unsafe impl Foo for i32 {
    // method implementations go here
}
```

