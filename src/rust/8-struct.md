![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 结构体：强大而灵活的数据类型


# 一、Rust 结构体概述

## （一）结构体的定义与实例化

在 Rust 中，使用 struct 关键字来定义结构体。

结构体包括：结构体名字和结构体内的数据名字及类型，这些被称为字段：
```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}
```
创建结构体实例，需要以结构体的名字开头，接着在大括号中使用 key: value 键值对的形式提供字段的值。

实例中字段的顺序不需要和结构体中声明的顺序一致。
```rust
let user = User {
    active: true,
    username: String::from("suntiger"),
    email: String::from("suntiger@example.com"),
    sign_in_count: 1,
};
```
## （二）结构体的可变性与所有权

在 Rust 中，结构体的可变性规则是默认不可变的。

如果要修改结构体的字段，需要将结构体声明为可变的，使用 mut 关键字。

一旦结构体实例是可变的，那么实例中所有的字段都是可变的。

结构体必须掌握字段值所有权，因为结构体失效的时候会释放所有字段。

比如，如果结构体中有字符串类型的字段，通常使用 String 而不是 &str，以确保结构体拥有对字符串数据的所有权。

## （三）结构体的三种类型

- 单元结构体：没有任何字段的结构体，被称为类单元结构体。它常常在需要在某个类型上实现 trait 但不需要在类型中存储数据的时候发挥作用。
```rust
struct OneUnit;
```
- 元组结构体：有结构体名称，但没有具体的字段名，只有字段的类型。当需要给整个元组取一个名字，并使元组成为与其他元组不同的类型时，元组结构体很有用。
```rust
struct Bounds(usize, usize);

let image_bounds = Bounds(1024, 768);

assert_eq!(image_bounds.0 * image_bounds.1, 786432);
```
- 具名结构体：最常见的结构体类型，每个字段都有具体的名称和类型。这种结构体使得数据的访问和操作更加清晰和方便。
```rust
struct GrayscaleMap {
    pixels: Vec,

    size: (usize, usize)
}
```
# 二、Rust 结构体的关键特性

## （一）泛型结构体

泛型结构体使代码更加通用和可复用。

通过使用泛型参数，我们可以创建具有通用类型的结构体，提高代码的可复用性。
```rust
struct Pair<T, U> {
    first: T,
    second: U,
}
```
这里，Pair是一个泛型结构体，它有两个泛型参数T和U，分别代表结构体中的第一个字段和第二个字段的类型。

我们还可以对泛型参数进行约束，以限制可接受的类型。比如：
```rust
trait Printable {
    fn print(&self);
}
impl Printable for i32 {
    fn print(&self) {
        println!("Value: {}", *self);
    }
}
impl Printable for String {
    fn print(&self) {
        println!("Value: {}", *self);
    }
}

struct Pair<T: Printable, U: Printable> {
    first: T,
    second: U,
}
```
在这个例子中，我们对泛型参数T和U进行了约束，它们必须实现Printable trait。

这样，就可以在代码中调用Pair结构体实例的print方法，并打印值。

## （二）生命周期结构体

在 Rust 中，结构体中引用的生命周期是一个重要的概念。

如果结构体的成员变量是一个外部对象的借用，那么必须标识这个借用对象的生命周期。
```rust
struct Piece<'a> {
    slice: &'a [u8],
}
```
这里，'a表示vec的生命周期，下面的例子调用，vec的生命周期至少应该大于等于piece的生命周期。
```rust
fn test() {
    let vec = Vec::<u8>::new();
    let piece = Piece {
        slice: vec.as_slice()
    };
}
```
如果有两个不同的成员，分别持有外部对象的借用，那么他们应该使用一个生命周期标识还是两个呢？
```rust
struct Piece<'a> {
    slice_1: &'a [u8],
    slice_2: &'a [u8],
}
```
这里，'a只是表示slice_1和slice_2所借用的对象的存活范围在一个相同的作用域内，而不是说slice_1和slice_2所借用的对象必须是同一个。

## （三）关联常量

在 Rust 中，结构体可以有关联常量。

关联常量是与类型本身相关联的常量，可以在结构体的定义中使用const关键字来定义。
```rust
struct Struct;
impl Struct {
    const ID: u32 = 0;
}

fn main() {
    println!("the ID of Struct is: {}", Struct::ID);
}
```
这里，ID是与Struct相关联的常量。关联常量可以在类型中提供一些固定的值，方便在代码中使用。

## （四）用 impl 定义方法

在 Rust 中，可以使用impl为结构体定义方法。和关联函数。

关联函数是与结构体相关联的函数，但不操作结构体的数据，也就不用接收&self作为参数，类似于其他语言的静态方法。

关联函数的调用使用双冒号::。
```rust
struct Point {
    x: u32,
    y: u32,
}

impl Point {
    // 构造方法
    fn new(x: u32, y: u32) -> Point {
        Point { x, y }
    }
}

fn main() {
    let p = Point::new(10, 10);
    println!("{:?}", p);
}
```
方法是在impl块中定义的与结构体实例相关联的函数，可以接收&self、&mut self或self作为参数，用于操作结构体的数据。
```rust

struct Point {
    x: u32,
    y: u32,
}

impl Point {
    // 类似于面向对象编程中对象的实例方法
    // 该方法返回 Point 结构体的 x 坐标
    fn get_x(&self) -> u32 {
        self.x
    }
    // 修改结构体中的数据，需要使用可变引用 &mut
    // 该方法修改 Point 结构体的 x 坐标
    fn set_x(&mut self, x: u32) {
        self.x = x;
    }
}
```
# 三、Rust 结构体的应用场景

## （一）数据封装与组织

- 通过将相关的数据组合在一个结构体中，可以清晰地表达数据的含义和关系，提高代码的可读性。

    例如，在一个图形绘制程序中，可以定义一个Rectangle结构体来表示矩形，包含width（宽度）和height（高度）两个字段。这样，在处理矩形相关的数据时，代码更加直观，易于理解。

- 同时，结构体也有助于提高代码的可维护性。当需要修改数据的表示或处理方式时，只需要在结构体的定义和相关方法中进行修改，而不会影响到其他不相关的部分。

    例如，如果要改变矩形的表示方式，从使用宽度和高度两个独立的字段，改为使用对角线的两个端点坐标，只需要在Rectangle结构体的定义和相关方法中进行修改，而不会影响到程序的其他部分。

## （二）类型安全与抽象

- 结构体可以提供强大的类型安全。

    例如，可以定义一个UserId结构体和一个OrderId结构体，即使它们内部可能都是存储一个整数类型，但通过不同的结构体类型，可以确保在代码中不会混淆用户 ID 和订单 ID。

- 通过自定义结构体类型，还可以实现抽象。

    例如，可以定义一个Shape结构体，然后为不同的形状（如圆形、矩形、三角形等）实现这个结构体。这样，在处理不同形状的图形时，可以通过统一的接口（即Shape结构体的方法）进行操作，而无需关心具体的形状类型。这种抽象可以提高代码的可扩展性和可维护性。

## （三）代码复用与扩展

- 结构体可以促进代码复用。

    例如，定义一个通用的Point结构体表示二维平面上的点，包含x和y两个字段。这个结构体可以在多个不同的项目或模块中复用，例如在图形绘制、地理信息系统等领域。

- 通过扩展结构体，可以实现新的功能。

    例如，可以使用泛型结构体来实现一个通用的容器类型，然后通过为特定类型实现特定的方法来扩展其功能。
