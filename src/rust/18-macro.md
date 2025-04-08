![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

#  Rust 宏全解析：从基础到实战，掌握代码生成魔法

## 一、宏基础：代码生成的编译器魔法

### 1.1 宏是什么？
宏（Macro）是 Rust 编程语言的重要组成部分。它是一种扩展编译器功能的方法，从而可以支持标准之外的功能。

从根本上说，宏是一种编写代码来生成其他代码的方法，被称为 元编程（Metaprogramming）。

宏允许你在编译阶段生成代码，通过`macro_rules!`或`proc_macro`来定义，打破了常规代码的静态限制，让你的代码在编译时就能根据特定规则进行扩展和变化。

### 1.2 宏的作用

**减少重复代码**：宏通过展开来生成代码，可以明显减少重复的代码，并且增加了代码的可维护性。

**生成复杂代码**：宏可以根据输入生成复杂的代码结构，这对于一些需要动态生成代码的场景非常有用。

**提高性能**：宏在编译阶段展开，所以它们的执行效率通常比函数调用更高。

**灵活性**：宏的另一个优势是非常灵活，因为它们可以接收动态数量的参数或输入，而函数则不行。

### 1.3 宏与函数的区别



| 特性     | 函数        | 宏                           |
| ------ | --------- | --------------------------- |
| 执行时机   | 运行时       | 编译时                         |
| 参数处理   | 固定类型 / 数量 | 可变类型 / 数量                   |
| 代码生成能力 | 无法生成新结构   | 可生成任意 Rust 代码               |
| 调用方式   | 直接使用函数名调用 | 使用宏名加感叹号调用，如`macro_name!()` |

### 1.4 宏展开示例

下面来看一个简单的宏展开示例，通过`macro_rules!`定义一个加法宏`add`：

```rust
macro_rules! add {
   ($a:expr, $b:expr) => {
       $a + $b
   };
}

fn main() {
   let result = add!(1, 2);
   println!("结果: {}", result);
}
```

在这个例子中，`add!`宏接收两个表达式作为参数，当宏被调用时，它会将`$a`和`$b`替换为传入的实际表达式，展开后的代码就相当于`let result = 1 + 2;`。这样，通过宏我们实现了一种灵活的代码生成方式，根据不同的输入生成不同的代码逻辑。

## 二、宏的分类：声明宏 vs 过程宏

Rust 宏主要分为声明宏（Declarative Macro）和过程宏（Procedural Macro）两大类别，其中过程宏又分为派生宏（Derive Macro）、属性宏（Attribute Macro）和类函数宏（Function-like Macro）

### 2.1 声明宏（Declarative Macro）

声明宏，就像是一个精巧的代码模板工厂，使用`macro_rules!`来定义，它基于模式匹配的原理，能够根据不同的输入模式生成相应的代码。

这种宏在处理一些相对简单但重复性高的代码生成任务时，表现得尤为出色。

来看一个简单的示例，定义一个`print_info`宏，用于打印各种类型的信息：

```rust
macro_rules! print_info {
   ($value:expr) => {
       println!("信息: {}", $value);
   };
}

fn main() {
   let name = "Rust";
   print_info!(name);
   let number = 42;
   print_info!(number);
}
```

在这个例子中，`print_info!`宏接受一个表达式参数`$value`，当宏被调用时，它会根据传入的参数，将`$value`替换到`println!`宏中，从而生成对应的打印代码。

### 2.2 过程宏（Procedural Macro）

过程宏通过`proc_macro`模块来处理语法树，能够对代码进行更复杂的操作和转换。过程宏又细分为三种类型，每一种都有着独特的 “魔法技能”。

#### 2.2.1 派生宏（Derive Macro）

派生宏是过程宏中的 “便捷助手”，主要用于自动为结构体或枚举生成`trait`实现。使用派生宏，就不用手动编写那些繁琐的`trait`实现代码，大大提高了开发效率。

以`Debug`派生宏为例，当我们为一个结构体加上`#[derive(Debug)]`属性时，编译器会自动为该结构体生成`Debug` trait 的实现代码，让我们可以方便地使用`println!("{:?}", instance)`来打印结构体的调试信息。

```Rust
#[derive(Debug)]
struct Point {
   x: i32,
   y: i32,
}

fn main() {
   let p = Point { x: 10, y: 20 };
   println!("{:?}", p);
}
```

在这个例子中，`#[derive(Debug)]`就是派生宏的应用，它自动为`Point`结构体生成了`Debug` trait 的实现，使我们能够轻松地打印出`Point`结构体的内部状态。

接下来实现一个简单的`HelloMacro`派生宏：
```rust

// crate hello_macro; macro lib
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

// 实现派生宏
#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // 解析输入的 TokenStream 为 DeriveInput 结构体
    let ast = parse_macro_input!(input as DeriveInput);

    // 获取结构体的名称
    let name = &ast.ident;

    // 使用 quote 生成代码
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };

    // 将生成的代码转换为 TokenStream 并返回
    gen.into()
}

// bin
// 定义一个trait
pub trait HelloMacro {
    fn hello_macro();
}

// 测试代码
#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}    
```
在这个示例中，我们定义了一个`HelloMacro`特性，然后通过`proc_macro_derive`宏为该特性实现了一个派生宏`hello_macro_derive`。其中`hello_macro_derive`名字不重要，重要的是`#[proc_macro_derive(HelloMacro)]`里面的`HelloMacro`，这才是编写的派生宏的真正名字；

当我们为一个结构体加上`#[derive(HelloMacro)]`属性时，编译器会自动为该结构体生成`HelloMacro`特性的实现代码，实现了一个简单的打印功能。
运行结果：
```Shell
Hello, Macro! My name is Pancakes!
```
在这个例子中，`#[derive(HelloMacro)]`就是派生宏的应用，它自动为`Pancakes`结构体生成了`HelloMacro`特性的实现，使我们能够轻松地调用`Pancakes::hello_macro()`来打印出`Pancakes`结构体的名称。  
通过这个例子，可以看到派生宏的强大之处，它能够自动生成代码，大大减少了重复劳动，提高了开发效率。

同时，派生宏也为我们提供了一种简洁、高效的方式来处理代码生成任务，让我们的编程体验更加愉悦。

#### 2.2.2 属性宏（Attribute Macro）

属性宏是代码世界里的 “魔法画笔”，可以为函数、模块等代码元素添加自定义属性，实现各种神奇的功能。比如，我们可以定义一个属性宏，用于记录函数的调用信息，方便调试和性能分析。

```Rust
// macro lib
use proc_macro::TokenStream;

use quote::quote;

use syn::{parse_macro_input, ItemFn};

#[proc_macro_attribute]
pub fn log_call(_attr: TokenStream, item: TokenStream) -> TokenStream {

   let input = parse_macro_input!(item as ItemFn);

   let name = input.sig.ident;

   let block = input.block;

   let expanded = quote! {

       fn #name() {
           println!("调用函数: {}", stringify!(#name));

           #block
           println!("函数 {} 执行完毕", stringify!(#name));
       }
   };
   expanded.into()
}

// bin

#[log_call]
fn greet() {
   println!("Hello, Rust!");
}

fn main() {
   greet();
}
```

在这个示例中，我们定义了一个`log_call`属性宏，当它应用到`greet`函数上时，会在函数的开头和结尾分别插入打印调用信息和执行完毕信息的代码，让我们对函数的执行过程一目了然。

#### 2.2.3 函数宏（Function-like Macro）

函数宏就像是一个灵活多变的 “代码生成器”，它类似于声明宏，但更加灵活，能够接受`TokenStream`作为输入，并返回一个新的`TokenStream`作为输出，常用于生成复杂的表达式。

```Rust
use proc_macro::TokenStream;

use quote::quote;

use syn::{parse_macro_input, Expr};

#[proc_macro]
pub fn square(input: TokenStream) -> TokenStream {
   let input_expr = parse_macro_input!(input as Expr);

   let output = quote! {
       #input_expr * #input_expr
   };
   output.into()
}
```

在这个例子中，`square!`宏接受一个表达式作为参数，然后返回该表达式的平方，为我们提供了一种简洁的方式来生成复杂的数学表达式。

## 三、内建宏：Rust 自带的代码生成神器

Rust 内建宏（Built-in Macros）是一系列预先定义好的强大工具，这些内建宏就像是一群默默奉献的小助手，在你编写代码的过程中，为你简化各种常见的操作，让你的编程之旅更加轻松愉快。

### 3.1 常用内建宏



| 宏              | 用途                      | 示例                                           |
| -------------- | ----------------------- | -------------------------------------------- |
| `stringify!`         | 将表达式转换为字符串字面量                  | `stringify!(2 + 3); // 输出字条串"2 + 3"`                     |
| `concat!`         |  将多个字符串字面量连接成一个单一的字符串字面量                 | `concat!("Hello", ", ", "World!");`                     |
| `cfg!`         |  编译时的布尔宏，对编译配置进行检查                 | `cfg!(target_os = "windows") // 返回布尔值`                     |
| `env!`         |  编译时宏，用于在编译期间获取环境变量的值                 | `let database_url = env!("DATABASE_URL");`                     |
| `vec!`         | 创建动态数组                  | `let v = vec![1, 2, 3];`                     |
| `assert!`      | 测试断言                    | `assert!(2 + 2 == 4);`                       |
| `println!`     | 格式化字符串并输出到标准输出          | `println!("Hello, {}!", "Rust");`            |
| `include!` | 编译时宏，将一个文件的内容插入到当前文件中进行编译。 | `include!("shared_code.rs");` |
| `include_str!` | 嵌入文件内容，将文件内容作为字符串引入到代码中 | `let content = include_str!("example.txt");` |

### 3.2 derive 系列宏

`derive`系列宏就像是一位贴心的代码生成助手，它能够自动为结构体或枚举生成一些常用`trait`的实现代码，大大减少了我们手动编写代码的工作量。

**Debug**：这个宏就像是给你的结构体或枚举配备了一个 “调试放大镜”，它会自动生成`fmt`方法，让你可以方便地使用`println!("{:?}", instance)`来打印实例的详细调试信息，帮助你快速了解实例的内部状态。

```Rust
#[derive(Debug)]
struct Point {
   x: i32,
   y: i32,
}

fn main() {
   let p = Point { x: 10, y: 20 };
   println!("{:?}", p);
}
```

**Clone**：`Clone`宏就像是一个神奇的 “克隆机器”，它会为你的结构体或枚举生成`clone`方法，让你可以轻松地复制实例，而无需手动编写复杂的复制逻辑。

```Rust
#[derive(Clone)]
struct Person {
   name: String,
   age: u32,
}

fn main() {
   let person = Person { name: String::from("Alice"), age: 25 };
   let cloned_person = person.clone();
   println!("Original: {}, {}", person.name, person.age);
   println!("Clone: {}, {}", cloned_person.name, cloned_person.age);
}
```

**Hash**：`Hash`宏为你的类型生成哈希函数，就像是给每个实例分配了一个独特的 “指纹”，方便在哈希表等数据结构中进行高效的查找和比较。

```Rust
#[derive(Hash)]
struct Book {
   title: String,
   author: String,
}

let book1 = Book { title: String::from("The Rust Programming Language"), author: String::from("Steve Klabnik") };

let book2 = Book { title: String::from("Another Book"), author: String::from("Another Author") };

let mut map = std::collections::HashMap::new();

map.insert(book1, 1);

map.insert(book2, 2);
```

**PartialEq**：这个宏会生成部分相等比较的方法，让你可以使用`==`和`!=`运算符来比较两个实例是否相等，就像是给你的类型赋予了一把 “比较尺子”。

```Rust
#[derive(PartialEq)]
struct Rectangle {
   width: u32,
   height: u32,
}

let rect1 = Rectangle { width: 10, height: 20 };

let rect2 = Rectangle { width: 10, height: 20 };

let rect3 = Rectangle { width: 15, height: 25 };

assert!(rect1 == rect2);

assert!(rect1 != rect3);
```

### 3.3 调试宏

调试宏具有很大挑战性，最大的问题是在宏展开的过程中缺少可视性。

Rust 提供了三个工具来帮助你调试宏。

注意：这些特性都是unstable的，因为它们被设计为用在开发的过程中，而不是最后的代码中;

1、使用cargo build --verbose 来查看Cargo是如何调用rustc 的命令，然后拷贝rustc 的命令行并加上-Z unstable-options --pretty expanded 选项。完全展开后的代码会输出到终端。然而只有当你的代码没有语法错误时这种方式才能生效。

2、Rust 提供了一个log_syntax!() 宏简单地在编译期把它的参数打印到终端。这个宏需要#![feature(log_syntax)] 特性标记。

3、让Rust编译器把所有宏调用输出到终端。在代码中插入trace_macros!(true)，
之后每当Rust展开一个宏时，它都会打印出宏的名字和参数。
参考如下代码：

```Rust
#![feature(trace_macros)]

fn main() {
    trace_macros!(true);
    let numbers = vec![1, 2, 3];
    trace_macros!(false);
    println!("total: {}", numbers.iter().sum::<u64>());
}
```
这个代码会输出如下：
```
$ rustup override set nightly
...
$ rustc trace_example.rs
note: trace_macro
--> trace_example.rs:5:19
|
5 | let numbers = vec![1, 2, 3];
| ^^^^^^^^^^^^^
|
= note: expanding `vec! { 1 , 2 , 3 }`
= note: to `< [ _ ] > :: into_vec ( box [ 1 , 2 , 3 ] )`
```
编译器会显示每一个宏调用的代码，包括展开之前和展开之后的代码。trace_macros!(false);
这一行关闭了追踪，因此println!() 的调用不会被追踪。

## 四、实战演练：实现自定义 json! 宏

理论知识储备完毕，接下来让我们进入实战环节，亲手打造一个属于自己的`json!`宏，来深入体验 Rust 宏的强大威力。

### 4.1 定义 JSON 枚举

首先，定义一个枚举来表示 JSON 数据的各种类型。

这个枚举就像是一个万能的容器，能够容纳 JSON 中的各种值，无论是简单的数字、字符串，还是复杂的数组和对象。

```Rust
use std::collections::HashMap;

#[derive(Clone, PartialEq, Debug)]
pub enum Json {
   Null,
   Boolean(bool),
   Number(f64),
   String(String),
   Array(Vec<Json>),
   Object(HashMap<String, Json>),
}
```

在这个枚举中，`Null`表示 JSON 中的`null`值，`Boolean`表示布尔值，`Number`表示数字，`String`表示字符串，`Array`表示数组，`Object`表示对象。

通过这个枚举，我们可以将 JSON 数据转化为 Rust 中的数据结构，方便后续的处理和操作。

### 4.2 实现 json! 宏

接下来，就是见证奇迹的时刻，我们要实现`json!`宏，让它能够将 JSON 风格的语法转换为我们定义的`Json`枚举。

由于json宏需要支持json!(true)、json!(1)、json!(1.0)、json!("yes")等多种类型，如果是一个一个的匹配，将是非常繁琐且麻烦：
```Rust
macro_rules! json {
(true) => {
    Json::Boolean(true)
};
(false) => {
    Json::Boolean(false)
};
...
}
```
Rust 有一种标准的trait, 可以把多种类型的值转换成另一种特定类型的方法：From trait。因此可以简单地为几种类型实现这个trait：

```Rust
impl From<bool> for Json {
    fn from(b: bool) -> Json {
        Json::Boolean(b)
    }
}
impl From<i32> for Json {
    fn from(i: i32) -> Json {
        Json::Number(i as f64)
    }
}
impl From<String> for Json {
    fn from(s: String) -> Json {
        Json::String(s)
    }
}
impl<'a> From<&'a str> for Json {
    fn from(s: &'a str) -> Json {
        Json::String(s.to_string())
    }
}
```

但是，所有的12种数字类型应该有非常相似的实现，因此编写一个宏来批量为数字类型实现From trait：
```Rust
macro_rules! impl_from_num_for_json {
    ( $( $t:ident )* ) => {
        $(
            impl From<$t> for Json {
                fn from(n: $t) -> Json {
                    Json::Number(n as f64)
                }
            }
        )*
    };
}
impl_from_num_for_json!(u8 i8 u16 i16 u32 i32 u64 i64 u128 i128
    usize isize f32 f64);

```

完整的json宏如下：

```Rust

#[derive(Clone, PartialEq, Debug)]
pub enum Json {
    Null,
    Boolean(bool),
    Number(f64),
    String(String),
    Array(Vec<Json>),
    Object(HashMap<String, Json>),
}

// 为数字类型实现 From trait的声明宏
macro_rules! impl_from_num_for_json {
    ( $( $t:ident )* ) => {
        $(
            impl From<$t> for Json {
                fn from(n: $t) -> Json {
                    Json::Number(n as f64)
                }
            }
        )*
    };
}

impl_from_num_for_json!(u8 i8 u16 i16 u32 i32 u64 i64 u128 i128
    usize isize f32 f64);

impl From<bool> for Json {
    fn from(b: bool) -> Json {
        Json::Boolean(b)
    }
}

impl From<String> for Json {
    fn from(s: String) -> Json {
        Json::String(s)
    }
}

impl<'a> From<&'a str> for Json {
    fn from(s: &'a str) -> Json {
        Json::String(s.to_string())
    }
}


macro_rules! json {
    (null) => {
        Json::Null
    };
    ([ $( $element:tt),* ]) => {
        Json::Array(vec![ $( json!($element) ),* ])
    };
    ({ $( $key:tt : $value:tt ),* }) => {
        Json::Object(vec![
            $( ($key.to_string(), json!($value)) ),*
        ].into_iter().collect())
    };
    ( $other:tt ) => {
        Json::from($other) // 处理布尔/数字/字符串
    };
}
```

这个宏的实现看起来有点复杂，但其实思路很清晰。它通过模式匹配来识别不同类型的 JSON 数据，并根据匹配结果生成相应的`Json`枚举值。

比如：
- 当匹配到`null`时，就返回`Json::Null`；
- 当匹配到数字时，根据数字的类型进行转换并返回`Json::Number`；
- 当匹配到字符串时，将字符串转换为`Json::String`；
- 当匹配到数组时，递归地调用`json!`宏来处理数组中的每个元素，并将结果存储在`Json::Array`中；
- 当匹配到对象时，将键值对插入到`HashMap`中，并返回`Json::Object`。
- 当匹配到`true`或`false`、数字、字符串时，就调用`Json::from`返回对应的值；

### 4.3 使用示例

现在，我们已经成功实现了`json!`宏，让我们来看看如何使用它。就像魔法师挥动魔杖一样，我们只需要在代码中调用`json!`宏，传入 JSON 风格的参数，就能轻松地创建出 JSON 数据。

```Rust
fn main() {
   let json_obj = json!({
       "name": "Rust",
       "version": 1.0,
       "features": ["memory safety", "concurrency", "performance"],
       "is_great": true
   });

   println!("{:?}", json_obj);
}
```

在这个例子中，我们使用`json!`宏创建了一个 JSON 对象，包含了`name`、`version`、`features`和`is_great`四个字段。通过`println!("{:?}", json_obj);`语句，我们可以将这个 JSON 对象打印出来，查看其内部结构。运行这段代码，你会看到终端输出如下结果：

```Rust
Object({
   "name": String("Rust"),
   "version": Number(1.0),
   "features": Array([
       String("memory safety"),
       String("concurrency"),
       String("performance"),
   ]),
   "is_great": Boolean(true),
})
```

从输出结果可以看出，`json!`宏成功地将我们传入的 JSON 风格的参数转换为了`Json`枚举，并且结构清晰，易于理解。
