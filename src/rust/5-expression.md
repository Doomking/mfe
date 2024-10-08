![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 表达式：强大而独特的编程利器


# 一、Rust 表达式综述

Rust 表达式在编程中占据重要地位，多数内容都是表达式，强大且功能丰富。

**Rust 是一种表达式语言**，遵循古老的传统，就像 Lisp 一样，**表达式负责完成所有工作**。

在 C 语言中，表达式和语句有明显区别，表达式有值，而语句没有。

但在 Rust 中，**if 和 match 等控制流工具都是表达式，可以生成值**。

例如，在初始化变量时可以使用 if 表达式：

```rust
let status = if cpu.temperature <= MAX_TEMP {
    HttpStatus::Ok
 } else {
     HttpStatus::ServerError
 };
```

match 表达式也可以作为参数传给函数或宏，如
```rust
println!("Inside the vat, you see {}.", match vat.contents {
    Some(brain) => brain.desc(),
    None => "nothing of interest"
});

```

Rust 的表达式类型丰富多样，包括无块表达式和带块表达式。

*   无块表达式可以是字面量表达式、路径表达式、运算符表达式等多种形式；
*   带块表达式则包括块表达式、不安全块表达式、循环表达式、If 表达式、If Let 表达式、Match 表达式等。

# 二、特点多样的表达式

## （一）强大的类型分类

Rust 表达式分为左值和右值两类。

- 左值表达式，又叫位置表达式，可以表达一个内存地址，能放到赋值运算符左边使用。
- 右值表达式，又叫值表达式，引用了某个存储单元地址中的数据，只能进行读操作。

Rust 表达式类型丰富多样，包含运算表达式、字面量表达式、方法调用表达式、数组表达式、索引表达式、单目运算符表达式、双目运算符表达式、语句块表达式等。
这种丰富的类型使得 Rust 表达式功能强大，并且每种表达式都可以嵌入到另外一种表达式中，组成更强大的表达式，体现了 Rust 表达式语法具有非常好的“一致性”。

## （二）运算表达式丰富

Rust 的运算表达式涵盖了算术运算符、比较运算符、位运算符、逻辑运算符等。

### 1. 算术运算符

Rust 的算术运算符包括加（+）、减（-）、乘（\*）、除（/）、求余（%）。例如：
```rust
fn main() {
    let x = 100;
    let y = 10;
    println!("{} {} {} {} {}", x + y, x - y, x \* y, x / y, x % y);
}
```


这段代码中，分别对变量 x 和 y 进行了加法、减法、乘法、除法和求余运算，并打印出结果。

### 2. 比较运算符

Rust 的比较运算符包括等于（==）、不等于（!=）、小于（<）、大于（>）、小于等于（<=）、大于等于（>=）。比较运算符的两边必须是同类型的，并满足 PartialEq 约束，比较表达式的类型是 bool。比如：
```rust
fn main() {
    let a = 1;
    let b = 10;
    let c = 10;
    let d = "string";
    println!("a!= b : {}", a!= b); println!("b == c : {}", b == c);
}
```


### 3.  位运算符

位运算符有按位与（&）、按位或（|）、按位异或（^）、左移（<<）、右移（>>）等。位运算符可以对二进制位进行操作，在特定的场景下非常有用。

### 4.  逻辑运算符

逻辑运算符包括逻辑与（&&）、逻辑或（||）、逻辑非（!）。逻辑运算符用于执行逻辑操作，返回布尔值。例如：
```rust
let a = true;
let b =!a;
println!("b: {}", b);
```


这里对布尔值 a 取逻辑非，将 true 变为 false，并打印结果。

# 三、优先级明确

Rust 的表达式优先级确保了复杂表达式的计算顺序。例如，在limit < 2 \* broom.size + 1中，**.运算符具有最高优先级**，因此会最先访问字段。

数组字面量、元组、分组等表达式类型具有较高优先级，而赋值、复合赋值等表达式类型优先级相对较低。

# 四、块与分号灵活运用

## （一）块作为通用表达式

块是一种最通用的表达式。一个块生成一个值，并且可以在任何需要值的地方使用。例如：
```rust
let display_name = match post.author() {
    Some(author) => author.name(),
    None => {
        let network_info = post.get_network_metadata()?;
        let ip = network_info.client_address();
        ip.to_string()
    }
};
```


在这个例子中，Some(author) =>之后的代码是简单表达式author.name()，None =>之后的代码是一个块表达式，它们对 Rust 来说没什么不同。块表达式的值是最后一个表达式ip.to\_string()的值。

## （二）分号的作用和结尾方式

大多数 Rust 代码行以分号或花括号结尾，就像 C 或 Java 一样。

在 Rust 中，分号确定块的返回值。如果省略分号，则返回值是块中最后一个表达式的值。在包含分号的情况下，返回值始终是 Rust 的单位值，即空元组()。例如：
```rust
let x = {
    let y = 1;
    y + 1
};
println!("x: {}", x);
```


这里x的值是块中最后一个表达式y + 1的值，即2。

如果在y + 1后面加上分号，那么x的值将是()。

赋值表达式也返回单位值，即(var = var + 1) == ()。

在这种特殊情况下，有没有分号在语义上可能没有差异，但包含分号会使代码的意图更加清晰，尤其是对于人类阅读代码来说。

# 五、if 与 match 的魅力

## （一）功能强大的 match

match 是 Rust 中一个非常强大的控制流运算符，它允许一个值与一系列模式进行比较，并根据匹配的模式执行相应的代码。

模式可以由字面量、变量、通配符等多种内容构成，每一个模式都是一个分支，程序根据匹配的模式执行相应的代码。

例如，定义一个枚举类型 Coin：
```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}
```

可以使用 match 来根据不同的硬币类型返回不同的美分值：
```rust
fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => {
            println!("Lucky penny!");
            1
        },
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
   }
}
```
match 还可以绑定到值的模式匹配臂的另一个有用功能是它们可以绑定到与模式匹配的部分值。

例如，定义一个包含 UsState 的枚举类型 Coin：
```rust
#[derive(Debug)]
enum UsState {
    Alabama,
    Alaska, // --snip--
}
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState),
}
```
可以在 match 表达式中绑定到 Coin::Quarter 的状态值：
```rust
fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {:?}!", state);
            25
        }
    }
}
```
match 还可以用于处理 Option 类型。例如，定义一个函数，对 Option 类型的值进行加一操作：
```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}
```
Rust 要求 match 模式匹配是穷尽式的，即必须穷举所有的可能性，否则会导致程序错误。

可以使用通配符“”放置在其他分支之后，通配符“”会匹配上面没有指定的所有可能的模式。例如：
```rust
let x: u8 = 1u8;
match x {
    1 => println!("one"),
    2 => println!("two"),
    _ => ()
}
```
## （二）简洁的 if let

if let 是 match 的语法糖，用于处理只关心一种匹配而忽略其他匹配的情况。

它更加简洁，代码更少缩进和模板代码，放弃了穷举的可能。

例如，对于只关心 Option 中值为 Some 的情况：
```rust
let config_max = Some(3u8);
if let Some(max) = config_max {
    println!("The maximum is configured to be {}", max);
}
```
if let 也可以搭配 else 使用，当模式不匹配时执行 else 块中的代码。例如：
```rust
let config_max = Some(8u8);
if let Some(3) = config_max {
    println!("three");
} else {
    println!("others");
}
```
if let 在处理 Option 或 Result 类型时非常有用。例如，对 Option 类型的值进行判断和处理：
```rust
let num1 = Box::new(Some(1i32));if let Some(num3) = \*num1 { let num4 = num3 + 2i32; println!("{} {} {}", num1.unwrap(), num3, num4);}
```
# 六、循环多样

## （一）loop 循环

loop 是 Rust 中的一种无限循环结构，它没有内置的终止条件，会一直执行循环体内的代码，直到遇到 break 语句为止。这对于需要无限循环或循环次数未知的情况非常有用。

基本结构如下：
```rust
loop {
    // 循环体内的代码
    // 使用 break 语句来退出循环
}
```
**特点：**

1.  无终止条件：loop 没有内置的终止条件，这意味着它会一直运行，直到显式地使用 break 语句来停止它。


2.  使用 break 退出：要终止 loop 循环，必须在循环体内使用 break 语句。


3.  使用 continue 跳过迭代：也可以使用 continue 语句来跳过当前迭代，并立即开始下一次迭代。

**示例：**
```rust
fn main() {
    let mut count = 0;
    loop {
        println!("Count: {}", count);
        count += 1;
        if count == 10 {
            break;
        }
     }
}
```
在这个示例中，将会数到 10 并打印出每个数字。当计数达到 10 时，使用 break 语句退出循环。

综合示例：
```rust
use std::io;
fn main() {
    let mut choice = 'y';
    loop {
        println!("Welcome to the game!");
        println!("Do you want to play? (y/n)");
        let mut input = String::new();
        io::stdin().read_line(&mut input).expect("Failed to read line");
        choice = input.trim().chars().next().unwrap\_or('n');
        if choice == 'y' {
            println!("Playing...");
        } else {
            println!("Exiting...");
            break;
        }
    }
}
```
在这个示例中，我们定义了一个变量 choice 来存储用户的输入。使用 loop 来重复询问用户是否想继续玩游戏。使用 std::io::stdin().read_line() 来读取用户的输入。使用 break 语句来结束循环，当用户输入 ‘n’ 时。

## （二）while 循环

基本结构：
```rust
while condition {
    // 当条件为 true 时执行的代码块
}
```
特点：

while 循环会在每次迭代前检查给定的条件。

如果条件为 true，则执行循环体内的代码。如果条件为 false，则不执行循环体内的代码，并且循环终止。

与 loop 相比，while 循环提供了内置的终止条件，这使得它适合于那些你知道循环应该执行多少次或何时应停止的情况。

示例：
```rust
fn main() {
    let mut n = 1;
    while n < 101 {
        if n % 15 == 0 {
            println!("fizzbuzz");
        } else if n % 3 == 0 {
            println!("fizz");
        } else if n % 5 == 0 {
            println!("buzz");
        } else {
            println!("{}", n);
        }
        n += 1;
    }
}
```
## （三）for 循环

for 循环是开发中使用最频繁的循环，尤其是需要迭代一个序列或迭代器时。对于集合中的每个元素，它都会执行一次循环体。

基本结构：
```rust
for element in collection {
    // 循环体
}
```
特点：

1.  安全高效：Rust 的 for 循环更加安全和高效，而且它不会产生越界的风险。

2.  基于迭代器：Rust 中的 for 循环被设计成基于迭代器的，这意味着它是用来遍历迭代器中的元素。

示例：
```rust
let numbers = [1, 2, 3, 4, 5];
for number in numbers {
    println!("Number: {}", number);
}
```
还可以使用范围（Range）进行遍历：
```rust
for i in 0..10 {
    println!("Value of i is: {}", i);
}
```
for 循环也支持并行化，以提高性能：
```rust
let numbers = [1, 2, 3, 4, 5];
for number in numbers.par_iter() {
    println!("Number: {}", number);
}
```
在这个例子中，我们使用 par_iter() 方法将迭代器转换为并行迭代器，然后执行循环。

注意，这可能会增加代码的复杂性，并需要更多的内存来管理并行任务。

因此，在使用并行化 for 循环时，请确保你的代码是可并行化的，并且考虑到性能和内存开销。

# 七、Rust 表达式综合优势

Rust 的表达式体系以其丰富的类型、明确的优先级、灵活的块与分号运用以及强大的控制流结构，在编程领域展现出了独特的优势。

然而，Rust 表达式也存在一定的复杂性。对于初学者来说，可能需要花费一定的时间来理解和掌握其丰富的类型、优先级规则以及各种控制流结构。但是，一旦掌握了这些知识，开发者就能够充分发挥 Rust 表达式的强大功能，编写出高效、安全、可靠的代码。

综上所述，Rust 表达式虽然具有一定的复杂性，但它的功能强大、灵活性高，值得开发者深入学习和应用。在实际编程中，合理运用 Rust 表达式可以提高代码的质量和性能，为开发高质量的软件提供有力的支持。
