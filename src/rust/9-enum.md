![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 的枚举与模式：灵活又安全的工具


# 一、Rust 枚举

## （一）枚举的定义

枚举允许自定义类型，取值范围只能取自**预定义的命名常量的集合**。

枚举就像是一个集合，其中的每个成员都是这个集合的一个特定值。

枚举适用于一个值有多种可能的情况。但必须使用模式匹配来安全地访问数据。

例如，一个Color 的类型，取值范围为Red、Orange、Yellow 等等。
```rust
enum Color {
    Red,
    Orange,
    Yellow,
}
```
在内存中，枚举值被存储为整数。可以为枚举值指定数值，否则，Rust 会从0 开始自动分配值。
```rust
enum HttpStatus {
    Ok = 200,
    NotModified = 304,
    NotFound = 404,
    ...
}
```
默认情况下，Rust 使用能容纳所有值的最小的内建整数类型来存储枚举。大多数情况下都是一个单独的字节，

将枚举转换为整数是允许的：
```rust
assert_eq!(HttpStatus::Ok as i32, 200);
```
然而，反过来把整数转换为枚举是不允许的。

枚举也和结构体一样可以拥有方法：
```rust
impl TimeUnit {
    /// 返回该时间单位的复数名词。
    fn plural(self) -> &'static str {
        match self {
            TimeUnit::Seconds => "seconds",
            TimeUnit::Minutes => "minutes",
            TimeUnit::Hours => "hours",
            TimeUnit::Days => "days",
            TimeUnit::Months => "months",
            TimeUnit::Years => "years",
        }
    }
    /// 返回该时间单位的单数名词。
    fn singular(self) -> &'static str {
        self.plural().trim_end_matches('s')
    }
}
```
## （二）枚举的赋值与取值

枚举的赋值非常直观，比如let red = Color::Red；就是将枚举值Red赋给变量red。

取值可以通过直接打印，如println!("{:?}", red);，也可以通过match语句来根据不同的枚举值执行不同的操作。
```rust
match some_enum_value {
    EnumVariant1 => // do something for EnumVariant1,
    EnumVariant2 => // do something for EnumVariant2,
    _ => // handle other cases if needed.
}
```
## （三）带有关联数据的枚举

带有关联数据的枚举可以让我们在枚举成员中关联特定的数据类型。

比如Message枚举类型，其中Move成员与一个包含x和y坐标的结构体关联，Write成员与一个字符串关联，ChangeColor成员与三个整数关联。通过模式匹配，我们可以方便地访问这些关联数据。例如：
```rust
enum Message {
    Quit,
    Move {
        x: i32,
        y: i32
    },
    Write(String),
    ChangeColor(u8, u8, u8),
}
fn process_message(message: Message) {
    match message {
        Message::Quit => println!("Quit"),
        Message::Move { x, y } => println!("Move to ({}, {})", x, y),
        Message::Write(text) => println!("Write: {}", text),
        Message::ChangeColor(r, g, b) => println!("Change color to RGB({}, {}, {})", r, g, b),
   }
}
```
Rust 有三种枚举值，没有数据的对应类单元结构体。元组的对应类元组结构体。结构体的对应有花括

号和命名字段的结构体。一个枚举可以同时有这三种值：
```rust
enum RelationshipStatus {
    Single,
    InARelationship,
    ItsComplicated(Option<String>),
    ItsExtremelyComplicated {
        car: DifferentialEquation,
        cdr: EarlyModernistPoem,
    },
}
```
## （四）使用 Option 枚举处理空值

Option枚举是 Rust 中用于处理可能为空的值的强大工具。在除法函数中，如果除数为 0，则返回None，否则返回Some并包含除法运算的结果。例如：
```rust
fn divide(dividend: f64, divisor: f64) -> Option<f64> {
    if divisor == 0.0 {
        None
    } else {
        Some(dividend / divisor)
    }
}
```
## （五）泛型枚举

枚举可以使用泛型，最典型的就是经常在标准库使用的Option和Result。泛型枚举的语法和泛型结构体的语法完全一致。
```rust
enum Option<T> {
    None,
    Some(T),
}
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```
以下用几行代码定义了一个可以存储任意数量的T 类型值的BinaryTree 类型：
```rust
// 一个`T`类型的有序集合
enum BinaryTree<T> {
    Empty,
    NonEmpty(Box<TreeNode<T>>),
}
// 二叉树的一部分
struct TreeNode<T> {
    element: T,
    left: BinaryTree<T>,
    right: BinaryTree<T>,
}
```
# 二、Rust 模式

## （一）模式的定义与使用场景

模式在 Rust 中是一种特殊的语法，主要用来匹配类型中的结构，无论类型是简单还是复杂。结合使用模式和 match 表达式以及其他结构可以提供更多对程序控制流的支配权。模式在 Rust 中有广泛的应用场景，包括 let、if let、while let、for循环、函数参数和 match等场景。

如果想要访问枚举里面的数据，是不允许直接访问的，比如 let a = Some(1)；是一个Option的枚举，但是不能直接使用a.0来访问里面的数据1，因为Option类型有可能是None，所以就需要使用模式匹配。
```rust
let a = Some(1);
match a {
    Some(b) => println!("{}}", b),
    _ => println!("None")
}
```
## （二）模式的种类

### 1.  字面量、变量、通配符模式：

字面量是诸如整数、浮点数、字符、字符串、布尔值等。它们可以直接作为模式，例如：
```rust
let a = 1;
match a {
    1 => println!("a is 1"),
    2 => println!("a is 2"),
    n => println!("a is other"),
}
```
如果需要一个匹配所有值的模式，但又不关心匹配到的值，你可以使用单个下划线_ 作为模式，也就是通配模式：
```rust
let a = 2;
match a {
    1..=5 => println!("a is bigger than 1 and smaller than 5"),
    _ => println!("other"),
}
let b = 'd';
match b {
    'a'..='z' => println!("a is bigger than a and smaller than z"),
    _ => println!("other"),
}
```
### 2.  结构体或元组模式：

常用于复杂数据类型，匹配数据的多个部分
```rust
struct Point(f32, f32);
let a = Point(1.2, 3.4);
match a {
    Point(1.2, y) => {},
    Point(x, y) => {},
}

struct Point3d {
    x: f32,
    y: f32,
    z: f32,
}
let b = Point3d {
    x: 1,
    y: 2,
    z: 3,
};
match b {
    Point3d { x: 1, y: 2, .. } => {},
    _ => {}
}
```
### 3.  数组或切片模式：

数组模式匹配数组,通常用来过滤出某些特殊值：
```rust
let hsl = [12, 211, 3]
match hsl {
    [_, _, 0] => [0, 0, 0],
    [_, _, 255] => [255, 255, 255],
    ...,
    _ => {}
}
```
切片模式与数组类似，但不同的是，切片的长度可以变化，切片模式中的.. 匹配任意数量的元素：
```rust
let names = &["a", "b", "c"];
match names {
    [] => { println!("Hello, nobody.") },
    [a] => { println!("Hello, {}.", a) },
    [a, b] => { println!("Hello, {} and {}.", a, b) },
    [a, .., b] => { println!("Hello, everyone from {} to {}.", a, b) }
}
```
### 4.  引用模式：

Rust 模式支持两种和引用有关的特性。ref 模式会借用被匹配的值，& 模式匹配引用：
```rust
struct Account {
    name: String,
    language: String,
    gender: String,
}
let account = Account {
    name: "tom".to_string(),
    language: "Chinese".to_string(),
    gender: "man".to_string(),
};
// 如果不加 ref ，编译报错，提示会出现所有权转移
match account {
    Account { ref name, ref language, .. } => {
        greet(name, language);
        show_settings(&account); // ok
    }，
    _ => {},
}
```
& 模式,一个以&开始的模式只能匹配引用：
```rust
struct Point3d {
    x: f32,
    y: f32,
    z: f32,
}
let b = Point3d {
    x: 1,
    y: 2,
    z: 3,
};

let a = &b;

match a {
    &Point3d { x, y, z } => {},
    _ => {}
}

let c = Some("c");
match c {
    Some(&d) => {},
    _ => {}
}


// ref与&一起使用
struct Engine {
    width: f32,
    height: f32,
    type: String,
}
struct Wheel {
    circle: f32,
    color: String,
}
struct Car {
    engine: Engine,
    wheel: Wheel,
}
let car = Car {
    engine: Engine {
        width: 100,
        height: 200,
        type: "f4".to_string(),
    },
    wheel: Wheel {
        circle: 50,
        color: "black".to_string(),
    },
}
let borrow_car = Some(&car);
match borrow_car {
    // Some(&Car { engine, .. }) error: can't move out of borrow
    Some(&Car { ref engine, .. }) => {},
    ...
    None => {}
}
```
### 5.  匹配守卫：
有时一个匹配分支还需要附加条件才能满足，可以在分支模式后添加条件：
```rust
let x = 4;
let y = false;
match x {
    4 if y => println!("yes"),
    n => println!("{}", n),
    _ => println!("no"),
}
```
### 6.  匹配多个模式：

使用 | 语法。例如：
```rust
let x = 4;
let y = false;
match x {
    4 | 5 => println!("yes"),
    _ => println!("no"),
}

let next_char = 'a';
// Rust（目前）不允许在模式中使用尾开区间,例如0..100。
match next_char {
    '0'..='9' => self.read_number(),
    'a'..='z' | 'A'..='Z' => self.read_word(),
    ' ' | '\t' | '\n' => self.skip_whitespace(),
    _ => self.handle_punctuation(),
}
```
### 7.  绑定和@模式：

x @ pattern 用给定的pattern 来匹配，匹配成功时它会创建单个变量x ，并把整个值移动或拷贝进去，而不是为匹配值的每一部分创建一个变量。
```rust
struct Point3d {
    x: f32,
    y: f32,
    z: f32,
}
let a = Point3d {
    x: 1,
    y: 2,
    z: 3,
};
match a {
    Point3d { x, y, z } => {
        some_fn(&Point3d { x, y, z});
    },
    _ => {}
}
// 绑定和@模式
match a {
    b @ Point3d { x, y, z } => {
        some_fn(&b);
    },
    _ => {}
}

// 与范围一起
let x = 1u32;
match x {
    e @ 1..=5 | e @ 10..=15 => println!("get:{}", e),
    _ => (),
}
```
