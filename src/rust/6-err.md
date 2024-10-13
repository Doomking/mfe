![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 错误处理：独特而灵活多样


# 一、Rust 错误处理概述

Rust 的错误处理方式主要包括 ：panic、Result 等；

**Result** 是一个枚举类型，有 Ok 和 Err 两个变体，分别表示成功和失败。

一般的错误使用Result 类型来处理，**Result 通常代表程序之外的东西引起的问题**，例如错误的输入、网络中断、权限问题等。

**panic** 是另一种错误，**一种永远不应该发生的错误**。

当程序遇到一些由程序自身的bug 导致的非常糟糕的事情时它会panic。例如：

• 数组访问越界

• 整数除以0

• 在值为Err 的Result 上调用.expect() 方法

• 断言失败

# 二、主要错误处理方式

## （一）Result 枚举类型

Result 是一个枚举类型，使用泛型参数，通常以 T 存储正常情况的返回值，E 存储错误情况的信息。

Result表示成功时返回一个字符串，失败时返回一个字符串切片作为错误信息。

Result 通常用于函数返回值，以明确表示函数执行可能成功或失败。

例如，读取文件的函数可能返回Result，如果文件读取成功，返回包含文件内容的Ok变体，否则返回包含错误信息的Err变体。
```rust
use std::fs::File;
use std::io::Error;
fn read_file(file_path: &str) -> Result<String, Error> {
    std::fs::read_to_string(file_path).map_err(|e| e)
}
fn main() {
    let result = read_file("non_existent_file.txt");
    match result {
        Ok(content) => println!("File content: {}", content),
        Err(e) => println!("Error reading file: {}", e),
    }
}
```
### 1.  常用方法介绍

- **err方法**：以Option 类型返回错误的值；

- **unwrap方法**：如果result 是成功的结果，会返回成功的值。如果result 是错误的结果，这个方法会panic；

- **unwrap_or(fallback)方法**：如果result 是成功的，返回成功的值。否则，返回fallback，丢弃错误的值；

- **unwrap_or_else(fallback_fn)方法**：和unwrap_or(fallback)类似，但不是直接传递fallback 值，而是传递一个函数或者闭包；

- **expect(message)方法**：和unwrap基本相同，可以提供一条panic 时打印的错误信息；

- **as_ref方法**：把一个Result 转换为Result<&T, &E>；

- **as_mut方法**：和as_ref类似，但借用可变引用。返回类型是Result<&mut T, &mut E>

## （二）Option 枚举类型

Option 也是枚举类型，内部只有 Some 和 None 两个值。

Some 变体包含一个值，表示存在某个值；

None 变体表示没有值。

```rust

let some_value: Option = Some(42); // 表示存在一个整数值 42，

let no_value: Option = None; // 表示没有值。
```

**与 Result 的区别**
- 从名称上理解：Result 意思是结果，一般表示成功或失败，所以它的枚举值会是 Ok 和 Err；Option意思是选项，侧重点不是成功失败，而是不同的选项（有选项则为 Some、无选项则为 None）;
- 在功能上：Result 主要用于表示操作可能成功或失败的情况，并且可以携带错误信息；Option 主要用于表示一个值可能存在或不存在的情况。

## （三）Panic 机制

### 1.  触发条件
- 显式触发 panic 的情况可以通过调用panic!宏实现。例如：panic!("这是一个显式触发的 panic");
- 隐式触发 panic 的情况包括数组越界、除零操作等。例如：let v = vec![1, 2, 3]; println!("{v[4]}");会因为数组越界触发 panic；

### 2.  panic 的处理模式

Rust 处理 panic 有两种模式：Unwind 和 Abort。

Unwind 模式会开始回溯，清理栈上的数据；

Abort 直接终止程序，不进行任何清理。

在 Cargo.toml 文件中，可以配置 panic 的处理方式，如[profile.release] panic = 'abort'设置为 abort 模式。

### 3.  捕获与恢复

使用std::panic::catch_unwind函数可以捕获和处理 panic：
```rust
use std::panic;
fn main() {
    let result = panic::catch_unwind(|| {
        panic!("发生 Panic");
     });
     match result {
         Ok(_) => println!("无 Panic"),
         Err(_) => println!("捕获到 Panic"),
     }
}
```
### 4.  与错误处理方式的关系

panic 用于处理不可恢复的错误，而 Result 和 Option 类型通常用于处理可预期的错误情况。

在实际开发中，更推荐使用 Result 枚举进行错误处理。

# 三、错误处理模式rust

## （一）有意不处理错误

### 1.  unwrap () 方法

在 Rust 中，unwrap()方法用于从Result类型中提取成功时的返回值。如果Result类型的值是Ok（表示成功），则unwrap()方法将返回内部的值；如果Result类型的值是Err（表示失败），则unwrap()方法将触发一个panic，抛出一个错误类型的错误。
```rust
let value: Option<i32> = Some(42); let result = value.unwrap(); // 这里会得到值 42
```
如果使用的是None：
```rust
let maybe_none: Option<i32> = None; ruslet result = maybe_none.unwrap(); // 这会导致编译错误，因为试图从 None 获取值
```
同样，在Result中，unwrap()仅适用于成功的值：
```rust
let ok_result: Result<i32, String> = Ok(42); rustlet result = ok_result.unwrap(); // 这里会得到值 42
```
然而，如果结果是Err：
```rust
let error_result: Result<i32, String> = Err("An error occurred"); let result = error_result.unwrap(); // 也会导致编译错误，因为这是错误的结果
```
unwrap()方法使用风险较大，可能会导致程序崩溃，因此应该谨慎使用，并且只应该在非生产的代码中使用。

### 2.  .fn()?符号

这个符号在 Rust 中的术语是 “提前返回选项”（early return option），作用等同于unwrap()。

只允许用于返回Result<>或Option<>类型的函数之后。

在 rust 的早期版本中，有个try!宏具有等效的功能。
```rust
use std::fs::File;
use std::io::Error;
fn open_file(file_path: &str) -> Result<File, Error> {
    let mut file = File::open(file_path)?; Ok(file)
}
```
这段函数内部使用File::open(file_path)?而演示代码中有意忽略了错误的情况。

## （二）自定义信息提示

使用expect()方法对错误做自定义信息提示，增强错误处理的可读性。

expect()方法在处理Result类型时，与unwrap返回一样。

将expect的参数作为传给panic!宏的错误信息，不像unwrap那样使用默认的panic值。
```rust
let ok_result: Result<i32, String> = Ok(42);
let result = ok_result.expect("自定义错误信息"); // 这里会得到值 42，如果是 Err 变体，会显示自定义错误信息并触发 panic
```
## （三）推荐处理方式

### 1.  match 处理

match处理Result或Option<>是一种推荐的错误处理方式。

通过match语句，可以明确地处理成功和失败的情况，提高代码的可读性和可维护性。

- 处理Result
```rust
let ok_result: Result<i32, String> = Ok(42);
match ok_result {
    Ok(value) => println!("成功：{}", value),
    Err(error) => println!("错误：{}", error),
}
```
- 处理Option：
```rust
let some_value: Option<i32> = Some(42);
match some_value {
    Some(value) => println!("有值：{}", value),
    None => println!("无值"),
}
```
### 2.map_err()方法

用于对Result中的错误进行映射，将一种错误类型转换为另一种错误类型，以便更好地处理错误。
```rust
let error_result: Result<i32, String> = Err("原始错误");
let mapped_result = error_result.map_err(|e| format!("映射后的错误：{}", e));
```
### 3. if let Some(value)= fn() {} else {}

这种方式在错误处理中非常简洁。如果函数返回Option类型的值，可以使用if let Some(value)= fn() {} else {}来处理有值和无值的情况。
```rust
let maybe_value: Option<i32> = Some(42);
if let Some(value) = maybe_value {
    println!("有值：{}", value);
} else {
    println!("无值");
}
```
### 4.使用特定函数

and_then()和or_else()在错误处理中也非常有用。and_then()用于在Result成功时执行一个函数，返回一个新的Result；or_else()用于在Result失败时执行一个函数，返回一个新的Result。
```rust
let ok_result: Result<i32, String> = Ok(42);
let new_result = ok_result.and_then(|value| Ok(value + 1));
```
-   or_else()示例：
```rust
let error_result: Result<i32, String> = Err("错误");
let new_result = error_result.or_else(|error| Ok(0));
```
在实际项目中，错误处理是一个非常重要的环节。我们应该根据不同的场景选择合适的错误处理方式，并遵循一些错误处理的最佳实践，以提高代码的健壮性和可维护性。
