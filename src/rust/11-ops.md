![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 运算符重载：开启高效编程之旅


# 一、算术和位运算符重载

## （一）二元算术运算

在Rust 中，表达式a + b 实际上是a.add(b) 的缩写，即对标准库中std::ops::Add trait 的add 方法的调用。

Rust 的标准数值类型都实现了std::ops::Add。

这是std::ops::Add 的定义：
```rust
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```
算术运算符重载是通过实现std::ops模块下的 trait 来实现的，如Add、Sub、Mul等。

以结构体Point为例，为其实现加法和减法运算符重载。
```rust
use std::ops::{Add, Sub};
#[derive(Debug, Copy, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}
impl Add for Point {
    type Output = Self;
    fn add(self, other: Self) -> Self {
        Self {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

impl Sub for Point {
    type Output = Self;
    fn sub(self, other: Self) -> Self {
        Self {
            x: self.x - other.x,
            y: self.y - other.y,
        }
    }
}
assert_eq!(Point { x: 3, y: 3 }, Point { x: 1, y: 0 } + Point { x: 2, y: 3 });
assert_eq!(Point { x: -1, y: -3 }, Point { x: 1, y: 0 } - Point { x: 2, y: 3 });
```
泛型的情况：
```rust
use std::ops::{Add, Sub};
#[derive(Debug, Copy, Clone, PartialEq)]
struct Point<T:Add<Output=T>> {
    x: T,
    y: T,
}

impl<T> Add for Point<T> where T: Add<Output=T> {
    type Output = Self;
    fn add(self, other: Self) -> Self {
        Self {
            x: self.x + other.x,
            y: self.y + other.y,
        }
     }
}

impl<T> Sub for Point<T> where T: Sub<Output=T> {
    type Output = Self;
    fn sub(self, other: Self) -> Self {
        Self {
            x: self.x - other.x,
            y: self.y - other.y,
        }
    }
}
assert_eq!(Point { x: 3, y: 3 }, Point { x: 1, y: 0 } + Point { x: 2, y: 3 });
assert_eq!(Point { x: -1, y: -3 }, Point { x: 1, y: 0 } - Point { x: 2, y: 3 });
```
考虑不同类型的Point+Point的情况：
```rust
use std::ops::{Add, Sub};
#[derive(Debug, Copy, Clone, PartialEq)]
struct Point<T:Add<Output=T>> {
    x: T,
    y: T,
}

impl<T,R> Add<Point<R>> for Point<T> where T: Add<R> {
    type Output = Point<T::Output>;
    fn add(self, other: Point<R>) -> Self::Output {
        Self {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

impl<T,R> Sub<Point<R>> for Point<T> where T: Sub<R> {
    type Output = Point<T::Output>;
    fn sub(self, other: Point<R>) -> Self::Output {
        Self {
            x: self.x - other.x,
            y: self.y - other.y,
        }
    }
}

assert_eq!(Point { x: 3, y: 3 }, Point { x: 1, y: 0 } + Point { x: 2, y: 3 });
assert_eq!(Point { x: -1, y: -3 }, Point { x: 1, y: 0 } - Point { x: 2, y: 3 });
```
## （二）一元运算符

考虑-x和!x的运算，-x 等价于x.neg()，!x 等价于 x.not()
```rust
trait Neg {
    type Output;
    fn neg(self) -> Self::Output;
}

trait Not {
    type Output;
    fn not(self) -> Self::Output;
}
```
为上述的Point实现求负和求反：
```rust
use std::ops::Neg;
impl<T> Neg for Point<T> where T: Neg<Output = T>, {
    type Output = Point<T>;
    fn neg(self) -> Point<T> {
        Point {
            x: -self.x,
            y: -self.y,
        }
    }
}
```
## （三）二元位运算

二元位运算包括&（与）、或（｜）、异或（^)、和移位<>

所有这些trait 都有相同的形式。

例如std::ops::BitXor(用于^ 运算符) 的定义：
```rust
trait BitXor<Rhs = Self> {
    type Output;
    fn bitxor(self, rhs: Rhs) -> Self::Output;
}
```
为Point实现异或：
```rust
use std::ops::BitXor;
impl<T> BitXor for Point<T> where T: BitXor<Output = T> {
    type Output = Point<T>;
    fn bitxor(self, rhs: Self) -> Point<T> {
        Point {
            x: self.x ^ rhs.x,
            y: self.y ^ rhs.y,
        }
    }
}
```
## （四）复合赋值运算符

复合赋值运算符例如x += y 或x &= y 需要两个参数，然后进行一些操作例如加法或位与，最后把结果保存到左侧的操作数。

在Rust 中，复合赋值表达式的值总是()，而不是最后被存储的值.

x += y 是方法调用x.add_assign(y) 的缩写，而add_assign 是std::ops::AddAssign trait 唯一的方法：
```rust
trait AddAssign<Rhs = Self> {
    fn add_assign(&mut self, rhs: Rhs);
}
```
为Point 类型实现AddAssign ：
```rust
use std::ops::AddAssign;
impl<T> AddAssign for Point<T> where T: AddAssign<T> {
    fn add_assign(&mut self, rhs: Point<T>) {
        self.x += rhs.x;
        self.y += rhs.y;
    }
}
```
# 二、相等性比较

Rust 的相等运算符== 和!=，是std::cmp::PartialEq trait 的eq 和ne 方法的缩写：

这是std::cmp::PartialEq 的定义：
```rust
trait PartialEq<Rhs = Self> where Rhs: ?Sized, {
    fn eq(&self, other: &Rhs) -> bool;
    fn ne(&self, other: &Rhs) -> bool {
        !self.eq(other)
    }
}
```
因为ne 方法有默认的定义，所以实现PartialEq 时只需要实现eq
```rust
impl<T: PartialEq> PartialEq for Point<T> {
    fn eq(&self, other: &Point<T>) -> bool {
        self.x == other.x && self.y == other.y
    }
}
```
相等性比较为什么使用PartialEq，即部分相等？

相等性是等价关系(equivalence relation) 的传统数学定义中的一种，等价关系需要满足三个要求。

对于任意值x和y：

1. 如果x == y 为真，那么y == x 也必须为真。交换等价性

2. 如果x == y 和y == z，那么x == z 也必须为真。相等性有传递性

2. x == x 必须总是为真

但是却是最后一个要求,导致问题变得复杂。

Rust 的f32 和f64 是IEEE 标准的浮点数类型。根据这个标准，像0.0/0.0 以及其他没有合适的结果的表达式必须产生一个特殊的非数(not-a-number) 值，通常被称为NaN 值。并且要求一个NaN 值必须和其他任何值都不相等——包括它自己。
```rust
assert!(f64::is_nan(0.0 / 0.0));
assert_eq!(0.0 / 0.0 == 0.0 / 0.0, false);
assert_eq!(0.0 / 0.0 != 0.0 / 0.0, true);
assert_eq!(0.0 / 0.0 < 0.0 / 0.0, false);
assert_eq!(0.0 / 0.0 > 0.0 / 0.0, false);
assert_eq!(0.0 / 0.0 <= 0.0 / 0.0, false);
assert_eq!(0.0 / 0.0 >= 0.0 / 0.0, false);
```
Rust 的== 运算符满足前两个等价关系的要求，因此被称为部分等价关系(partial equivalence relation)，因此Rust 使用名称PartialEq 作为内建的== 运算符。

如果泛型代码满足完全的等价关系，可以使用std::cmp::Eq trait 作为约束，它代表完全的等价关系。

如果一个类型实现了Eq，那么对于任何该类型的值x，x == x 一定为true。在实践中，几乎所有实现了PartialEq 的类型也实现了Eq，f32 和f64 是标准库中仅有的实现了PartialEq 但却没有实现Eq 的类型。

标准库将Eq 定义为PartialEq 的扩展，没有添加任何新方法：
```rust
trait Eq: PartialEq<Self> {}
```
如果类型是PartialEq 并且希望它也是Eq，那必须显式地实现Eq，即使并不需要为此再定义任何新的方法或类型：
```rust
impl<T: Eq> Eq for Point<T> {}
// 可以直接在Point 类型定义中的derive 属性里加上Eq：
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
struct Point<T> { ... }
```
# 三、顺序性比较

Rust 用单个trait std::cmp::PartialOrd 来指定比较运算符<, >, <=, >= 的行为：
```rust
trait PartialOrd<Rhs = Self>: PartialEq<Rhs> where Rhs: ?Sized, {
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;
    fn lt(&self, other: &Rhs) -> bool { ... }
    fn le(&self, other: &Rhs) -> bool { ... }
    fn gt(&self, other: &Rhs) -> bool { ... }
    fn ge(&self, other: &Rhs) -> bool { ... }
}
```
注意PartialOrd 扩展了PartialEq，只能对可以比较相等性的类型比较顺序性。

唯一需要实现的PartialOrd 的方法就是partial_cmp。当partial_cmp 返回Some(o) 时，o 表示self 和other 的关系：
```rust
enum Ordering {
    Less, // self < other
    Equal, // self == other
    Greater, // self > other
}
```
但如果partial_cmp 返回None，那么意味着self 和other 无法比较顺序。

在所有的Rust 基本类型中，只有浮点数的比较可能会返回None。

如果某个类型的两个值总是可以互相比较顺序性，那么可以实现更加严格的std::cmp::Ord trait：
```rust
trait Ord: Eq + PartialOrd<Self> {
    fn cmp(&self, other: &Self) -> Ordering;
}
```
cmp 方法直接返回Ordering，而不是像partial_cmp 一样返回Option，cmp 总是返回两个参数相等或它们的相对顺序。

# 四、Index 与 IndexMut

通过为类型实现std::ops::Index 和std::ops::IndexMut trait 来指明索引表达式例如a[i] 的行为。

数组直接支持[] 运算符，但对于任何其他类型，表达式a[i] 通常是*a.index(i) 的缩写，其中index 是std::ops::Index trait 的一个方法。

如果表达式被赋值或者可变借用，那么将是*a.index_mut(i) 的缩写，它是std::ops::IndexMut trait 的一个方法。

这是这两个trait 的定义：
```rust
trait Index<Idx> {
    type Output: ?Sized;
    fn index(&self, index: Idx) -> &Self::Output;
}

trait IndexMut<Idx>: Index<Idx> {
    fn index_mut(&mut self, index: Idx) -> &mut Self::Output;
}
```
可以用单个usize 值索引一个切片，来得到单个元素的引用，因为切片实现了Index。

也可以通过像a[i..j] 这样的表达式来引用一个子切片，因为它们也实现了Index>。
```rust
*a.index(std::ops::Range { start: i, end: j })
```
Rust 的HashMap 和BTreeMap 集合，可以用任何可哈希或可比较的类型作为索引。下面的代码能正常工作，因为HashMap<&str, i32> 实现了Index<&str>：
```rust
use std::collections::HashMap;
let mut m = HashMap::new();
m.insert(" 十", 10);
m.insert(" 百"， 100）；
m.insert(" 千", 1000);
m.insert(" 万"， 1_0000);
m.insert(" 億", 1_0000_0000);

assert_eq!(m[" 十"], 10);
assert_eq!(m[" 千"], 1000);
// 等价于
use std::ops::Index;
assert_eq!(*m.index(" 十"), 10);
assert_eq!(*m.index(" 千"), 1000);
```
