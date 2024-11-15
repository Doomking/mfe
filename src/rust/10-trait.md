![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 之 trait 与泛型的奥秘


# 一、trait 和泛型初印象

## （一）trait 的概念

trait 在 Rust 中类似于其他语言中的接口，可以作为接口使用、以参数的形式传入函数以及作为返回类型。与 C++ 的虚函数类似，都是对行为的抽象定义，但**Rust 没有类继承的概念**。

作为接口使用时，trait 把方法签名放在一起，定义了一组行为，不同的结构体可以实现同一个 trait，从而实现相同的行为。

例如用于写入字节的trait 叫做std::io::Write，它在标准库中的定义：
```rust
trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;
    fn write_all(&mut self, buf: &[u8]) -> Result<()> {
        ...
    }
    ...
}
```
标准类型File 和TcpStream 都实现了std::io::Write，Vec 也实现了。这三个类型都提供.write()、.flush() 等方法
```rust
use std::io::Write;
// out 的类型是&mut dyn Write，意思是“任何实现了Write trait 的值的可变引用”。
fn say_hello(out: &mut dyn Write) -> std::io::Result<()> {
    out.write_all(b"hello world\n")?;
    out.flush()
}

use std::fs::File;
let mut local_file = File::create("hello.txt")?;
say_hello(&mut local_file)?; // 可以工作
let mut bytes = vec![];
say_hello(&mut bytes)?; // 也可以工作
assert_eq!(bytes, b"hello world\n");
```
以**参数的形式**传入函数时，可以使用impl Article或者泛型写法fn notify(item_1: T)，这样可以接受实现了特定 trait 的类型作为参数，增强了函数的通用性。

作为**返回类型**时，只能返回确定的同一种类型，否则会报错。例如，pub fn notify(b: bool) -> impl Article，如果返回不同类型会导致编译错误。

## （二）泛型初体验

泛型是Rust 中另一种形式的多态。类似于C++ 的模板，一个泛型函数或类型可以用于多种不同的类型：
```rust
/// 给定两个值，找出较小的那个
fn min<T: Ord>(value1: T, value2: T) -> T {
    if value1 <= value2 {
        value1
    } else {
        value2
    }
}
```
泛型和trait 紧密相关：泛型函数在约束中使用trait 来表明它可以用于哪些类型的参数。

所以还会讨论&mut dyn Write 和 有哪些相似和不同之处，以及如何在这种两种使用trait 的方式中选择。

# 二、trait 详解

## （一）如何使用

一个trait 就是一个给定的类型可能支持也可能不支持的特性。

通常，一个trait 代表一种能力：一个类型可以做的事情。

有关trait 方法有一个不寻常的规则：trait **自身必须在作用域里**。否则，所有它的方法都会被隐藏：
```rust
let mut buf: Vec<u8> = vec![];
buf.write_all(b"hello")?; // 错误：没有叫`write_all`的方法
```
需要引入相应的trait
```rust
use std::io::Write;
let mut buf: Vec<u8> = vec![];
buf.write_all(b"hello")?; // ok
````
这个规则的原因是：可以用trait 来给任意类型添加新的方法——即使是标准库的类型例如u32 和str。

第三方的crate 也可以做同样的事情。但是如果不同的trait添加了**相同名字的方法**，这显然会导致名称冲突！

所以Rust 需要你自己显式的导入你需要使用的trait，这样rust就知道你使用的是哪个trait的方法了。

## （二）trait对象

rust中编写多态代码的方式之一就是trait对象。

rust中不允许直接定义类似 dyn Write的类型，因为一个变量的大小必须在编译期时已知，然而实现了Write 的类型可以是任何大小：
```rust
use std::io::Write;
let mut buf: Vec<u8> = vec![];
let writer: dyn Write = buf; // 错误：实现`Write`的类型有可能是任何大小，并没有固定的大小
```
但是Java等语言中可以直接定义一个接口类型的变量并赋值，原因就是这个变量是对一个**实现这个接口的对象的引用**。但是rust中想要实现引用的话，是需要显式引用的：
```rust
let mut buf: Vec<u8> = vec![];
let writer: &mut dyn Write = &mut buf; // ok
```
一个trait 类型的引用，例如writer，被称为一个trait 对象。和其他引用一样，一个trait 对象指向某个值、它有生命周期、它可以是可变的或者是共享的。

在内存中，一个trait 对象是一个胖指针，由指向值的指针加上一个指向表示该值类型的表的指针组成。因此每一个trait 对象要占两个机器字

## （三）泛型函数和类型参数

上面的say_hello()函数，以trait对象为参数，改成泛型参数，如下：
```rust
fn say_hello<W: Write>(out: &mut W) -> std::io::Result<()> {
    out.write_all(b"hello world\n")?;
    out.flush()
}
...
say_hello(&mut local_file)?; // 调用say_hello::<File>
say_hello(&mut bytes)?; // 调用say_hello::<Vec<u8>>
// 指明类型参数
say_hello::<File>(&mut local_file)?;
```
泛型函数可以有多个类型参数：
```rust
fn run_query<M: Mapper + Serialize, R: Reducer + Serialize>( data: &DataSet, map: M, reduce: R) -> Results {
    ...
}
// 泛型约束太长，可以使用Where
fn run_query<M, R>(data: &DataSet, map: M, reduce: R) -> Results
    where M: Mapper + Serialize, R: Reducer + Serialize {
    ...
}
```
一个泛型函数可以同时有生命周期参数和类型参数。生命周期参数在前：
```rust
fn nearest<'t,'c, P>(target: &'t P, candidates: &'c [P]) -> &'c P
    where P: MeasureDistance {
    ...
}
```
一个单独的方法也可以是泛型的，就算定义它的类型不是泛型的
```rust
impl PancakeStack {
    fn push<T: Topping>(&mut self, goop: T) -> PancakeResult<()> {
        goop.pour(&self); self.absorb_topping(goop)
    }
}
```
类型别名也可以是泛型的：
```rust
type PancakeResult<T> = Result<T, PancakeError>;
```
## （四）trait和泛型如何选择

### 1、trait对象

任何当你需要一个混合类型的值的集合的情况下trait 对象都是正确的选择
```rust
trait Vegetable {
    ...
}
// 只能是单一的蔬菜类型，不满足混合类型
struct Salad<V: Vegetable> {
    veggies: Vec<V>
}
// 以下类型没有固定大小，编译出错
struct Salad {
    veggies: Vec<dyn Vegetable> // 错误：`dyn Vegetable`并没有固定大小
}
// trait 对象可以实现混合类型
struct Salad {
    veggies: Vec<Box<dyn Vegetable>>
}
```
另一个使用trait 对象的可能的原因是：减小编译出的代码的体积，不像泛型，需要针对可能的类型编译成多个代码，导致体积过大；

### 2、泛型

#### （1）速度
   泛型函数签名中没有dyn 关键字。在编译期指明就确定了类型，不管是显式还是通过类型推导，编译器都知道用了哪个类型。没有使用dyn 关键字是因为没有trait 对象——因此也没有涉及动态分发。

#### （2）有的trait 不支持trait 对象
   trait 只支持一部分特性，例如**关联函数只能使用泛型**，这样就完全排除了trait 对象。

#### （3）很容易地一次给泛型类型参数添加多个trait 约束
   例如一个函数的参数T 就可以实现Debug + Hash + Eq。trait 对象不能这么做：Rust 不支持类似&mut (dyn Debug + Hash + Eq) 这样的类型。

# 三、实现trait

## （一）定义trait

通过trait关键字，给出trait名字和方法名即可：
```rust
/// 一个角色、物品、风景等
/// 任何可以显示在屏幕上的游戏世界的物体。
trait Visible {
    /// 在给定的画布上渲染这个对象。
    fn draw(&self, canvas: &mut Canvas);
    /// 如果点击(x, y) 会选中这个对象就返回true。
    fn hit_test(&self, x: i32, y: i32) -> bool;
}
```
## （二）实现trait

使用语法impl TraitName for Type：
```rust
impl Visible for Broom {
    fn draw(&self, canvas: &mut Canvas) {
        for y in self.y - self.height -1 .. self.y {
            canvas.write_at(self.x, y, '|');
        }
        canvas.write_at(self.x, self.y ,'M');
    }
    fn hit_test(&self, x: i32, y: i32) -> bool {
        self.x == x && self.y - self.height - 1 <= y && y <= self.y
    }
}
```
## （三）默认方法

trait中可以包括方法的默认实现，标准库中Write trait 的定义中包含了一个write_all 的默认实现
```rust
trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;
    fn write_all(&mut self, buf: &[8]) -> Result<()> {
        let mut bytes_written = 0;
        while bytes_written < buf.len() {
            bytes_written += self.write(&buf[bytes_written..])?;
        }
        Ok(())
    }
    ...
}
```
其中write 和flush 方法是每一个writer 必须实现的基本方法。一个writer 可能也实现了write_all，但如果没有，将会使用默认实现。

Rust 允许你在任意类型上实现任意trait，只要trait 或者类型是在当前crate 中定义的。
```rust
trait IsEmoji {
    fn is_emoji(&self) -> bool;
}
/// 为内建的字符类型实现IsEmoji 方法
impl IsEmoji for char {
    fn is_emoji(&self) -> bool {
        ...
    }
}
assert_eq!('$'.is_emoji(), false);
```
trait 中可以将Self 关键字用作类型。
```rust
pub trait Spliceable {
    fn splice(&self, other: &Self) -> Self;
}
/// Self是 CherryTree类型
impl Spliceable for CherryTree {
    fn splice(&self, other: &Self) -> Self {
        ...
    }
}
/// Self 是 Mammoth
impl Spliceable for Mammoth {
    fn splice(&self, other: &Self) -> Self {
        ...
    }
}
```
## （三）子trait

子trait类似于面向对象的子接口，但是不存在继承的关系
```rust
trait Creature: Visible {
    fn position(&self) -> (i32, i32);
    fn facing(&self) -> Direction;
    ...
}
```
任何实现Creature的类型，都必须实现Visible；

可以理解为Creature为Visible的子trait；其实就是约束关系；

子trait是对Slef的约束：
```rust
trait Creature where Self: Visible {
    fn position(&self) -> (i32, i32);
    fn facing(&self) -> Direction;
    ...
}
```
## （四）关联函数

trait 可以包含类型关联函数，Rust 中的关联函数类似于静态方法：
```rust
trait StringSet {
    /// 返回一个空的集合。
    fn new() -> Self;
    /// 返回一个包含`strings`中所有字符串的集合。
    fn from_slice(strings: &[&str]) -> Self;
    /// 查找集合是否包含`string`。
    fn contains(&self, string: &str) -> bool;
    /// 向集合中添加一个字符串。
    fn add(&mut self, string: &str);
}
```
## （四）完全限定调用方式

调用方法，可以使用使用完全限定方式：
```rust
"hello".to_string()
// 以下方式和上述的完全一致
str::to_string("hello")
ToString::to_string("hello")
<str as ToString>::to_string("hello")
```
当一个类型的多个trait有相同方法时，为了区分是调用哪个方法，就要使用完全限定调用：
```rust
outlaw.draw(); // error: draw on screen or draw pistol?
Visible::draw(&outlaw); // ok: draw on screen
HasPistol::draw(&outlaw); // ok: corral
```
当self 参数的类型不能被推断出来时
```rust
let zero = 0;
// 类型为定义：可能是`i8`，`u8`，...
zero.abs(); // 错误：不能在有歧义的数字类型
// 用类型调用方法`abs`
i64::abs(zero); // ok
```
当使用函数本身作为函数类型的值的时候：
```rust
let words: Vec<String> =
    line.split_whitespace()
    // 迭代器会产生&str 值
    .map(ToString::to_string) // ok .collect();
```
## （五）关联类型

看一下迭代器的例子：
```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    ...
}
```
其中Item就是一个关联类型，任何实现Iterator的类型，必须指明Item的类型，也就是迭代器迭代时返回的类型。

例如：
```rust
impl Iterator for Args {
    type Item = String;
    fn next(&mut self) -> Option<String> {
        ...
    }
    ...
}
// 或者
impl Iterator for Args {
    type Item = String;
    fn next(&mut self) -> Option<Self::Item> {
        ...
    }
    ...
}
```
通过关联类型就可以确保Iterator在迭代时可以返回正确的类型。

## （五）泛型类型

看一个乘法的例子：
```rust
/// std::ops::Mul，用于支持乘法(`*`) 的类型
pub trait Mul<RHS> {
    /// `*`运算符产生的结果的类型
    type Output;
    /// `*`运算符用到的的方法
    fn mul(self, rhs: RHS) -> Self::Output;
}
```
Mul 是一个泛型类型，类型参数RHS 是右手边(righthand side) 的缩写。

Mul 是一个泛型trait，可实例化出Mul、Mul、Mul 等不同的trait。

泛型trait 可以不受孤儿规则的约束：可以为一个外部类型实现一个外部trait，只要trait 的类型参数中有一个是在当前crate 中定义的类型。

## （六）impl Trait

通常情况下，可以使用trait对象来实现混合的类型，但是需要在调用时进行动态分发和堆分配，影响性能。

Rust 有一个专为此情形设计的特性叫做impl Trait。impl Trait 允许“擦除”返回值的类型，只指明它实现的trait 或traits，并且没有动态分发或者堆分配：
```rust
fn cyclical_zip(v: Vec<u8>, u: Vec<u8>) -> impl Iterator<Item=u8> {
    v.into_iter().chain(u.into_iter()).cycle()
}
```
这种方式属于静态分发，编译器需要在编译时就知道返回的类型，并编译成多个对应的类型，这样在运行时就没有开销；很重要的一点是要注意Rust **不允许trait 方法使用impl Trait 作为返回类型**。只有自由函数和关联类型函数可以使用impl Trait 作为返回值。

impl Trait 也可以用来在函数中接受**泛型参数**。
```rust
fn print<T: Display>(val: T) {
    println!("{}", val);
}
// 二者相同
fn print(val: impl Display) {
    println!("{}", val);
}
```
## （七）关联常量

trait 也可以有关联常量。
```rust
trait Greet {
    const GREETING: &'static str = "Hello";
    fn greet(&self) -> String;
}
// 可以声明常量但不赋值：
trait Float {
    const ZERO: Self;
    const ONE: Self;
}
impl Float for f32 {
    const ZERO: f32 = 0.0;
    const ONE: f32 = 1.0;
}
impl Float for f64 {
    const ZERO: f64 = 0.0;
    const ONE: f64 = 1.0;
}
// 你可以编写使用这些值的泛型代码：
fn add_one<T: Float + Add<Output=T>>(value: T) -> T {
    value + T::ONE
}
```
**关联常量不能和trait 对象一起使用**，因为编译器依赖实现的类型信息，才能在编译期找出正确的值。
