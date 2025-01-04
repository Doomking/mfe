![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 迭代器：从基础到实践


## 一 迭代器初印象

迭代器(iterator) 用于产生一个值的序列，通常会使用一个循环来进行处理。

Rust 中的迭代器基于两大关键特质：Iterator和IntoIterator。

### （一）Iterator

一个迭代器是任何实现了std::iter::Iterator trait 的类型：
```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    ... // 很多默认方法
}
```

Item 是迭代器产生的值的类型。next 方法可能返回Some(v)，其中v 是迭代器的下一个值；或者返回None，表示已经到达序列的终点。

### （二）IntoIterator

实现了std::iter::IntoIterator的类型，称为可迭代对象，它的into_iter 方法获取一个值并返回一个迭代它的迭代器：
```rust
trait IntoIterator where Self::IntoIter: Iterator<Item=Self::Item> {
    type Item;
    type IntoIter: Iterator;
    fn into_iter(self) -> Self::IntoIter;
}
```

for循环实际是对IntoIterator和Iterator的组合调用：
```rust
println!("There's:");
let v = vec!["antimony", "arsenic", "alumium", "selenium"];
for element in &v {
   println!("{}", element);
}
```

实际上，每一个for 循环只是IntoIterator 和Iterator 的方法调用的组合：
```rust
println!("There's:");
let v = vec!["antimony", "arsenic", "alumium", "selenium"];
let mut iterator = (&v).into_iter();
while let Some(element) = iterator.next() {
    println!("{}", element);
}
```

接受由itertor.next()产生的值的代码是迭代器消费者（consumer），上述的for循环代码就是消费者。

for 循环总是调用操作数的into_iter方法，但也可以直接向for 循环传递迭代器；例如，直接在Range 上循环时就是这种情况。所有的迭代器都会自动实现IntoIterator，它们的into_iter 方法简单地返回迭代器自身。

## 二 创建迭代器

### (一）iter 和iter_mut 方法

大多数集合类型提供iter 和iter_mut 方法，它们返回一个迭代器，迭代器会产生每一个item 的共享引用或可变引用。数组切片例如&[T] 和&mut [T] 也有iter 和iter_mut 方法。

除了使用for 循环自动处理之外，next方法是最常用的获得迭代器数据的方法：
```rust
let v = vec![4, 20, 12, 8, 6];
let mut iterator = v.iter();
assert_eq!(iterator.next(), Some(&4));
assert_eq!(iterator.next(), Some(&20));
assert_eq!(iterator.next(), Some(&12));
assert_eq!(iterator.next(), Some(&8));
assert_eq!(iterator.next(), Some(&6));
assert_eq!(iterator.next(), None);
```

如果某个类型有不止一种迭代方式，那么这个类型通常为每种遍历方式提供特定的方法，例如，&str 字符串切片类型没有iter 方法。作为替代，假设s 是&str，那么s.bytes() 返回一个产生s 的每个字节的迭代器，而s.chars() 会以UTF-8 编码解析它的内容，然后产生每一个Unicode 字符。

### （二）IntoIterator 实现

当一个类型实现了IntoIterator ，就可以调用它的into_iter 方法，获得一个迭代器，正如for 循环一样：
```rust
use std::collections::BTreeSet;

let mut favorites = BTreeSet::new();
favorites.insert("Lucy in the Sky With Diamonds".to_string());
favorites.insert("Liebesträume No. 3".to_string());
let mut it = favorites.into_iter();
assert_eq!(it.next(), Some("Liebesträume No. 3".to_string()));
assert_eq!(it.next(), Some("Lucy in the Sky With Diamonds".to_string()));
assert_eq!(it.next(), None);
```


大多数集合都提供了好几个IntoIterator 的实现，分别是为共享引用(&T)、可变引用(&mut T)、移动(T) 提供的实现：
```rust
or element in &collection { ... }
for element in &mut collection { ... }
for element in collection { ... }
```


切片实现了三种IntoIterator 变体中的两个；因为它们并不拥有自己引用的元素，因此没有“以值”的实现。

IntoIterator 产生共享和可变的引用，与调用iter 或者iter_mut 是等价的。

为什么Rust 同时提供两者？

1、IntoIterator 使for 循环能正常工作，因此是必要的；但当不使用for 循环时，使用favorites.iter() 比(&favorites).into_iter() 更加清晰。当频繁以共享引用方式迭代，iter 和iter_mut 就很有用。

2、IntoIterator 在泛型代码中很有用：可以使用一个约束，例如T: IntoIterator 来限制类型参数T 必须是可以迭代的类型。或者，可以写T: IntoIterator 来进一步要求迭代会产生U 类型的值。
```rust
use std::fmt::Debug;
fn dump<T, U>(t: T)
where T: IntoIterator<Item=U>, U: Debug
{
    for u in t {
        println!("{:?}", u);
    }
}
```

不能在泛型函数中使用iter 或者iter_mut，因为它们不是任何trait 的方法，大多数可迭代类型只是恰好有这两个方法。

### （三）from_fn 和successors

给定一个返回Option 的函数，std::iter::from_fn 返回一个迭代器，通过调用那个函数来产生item。
```rust
use rand::random; // 在Cargo.toml 中添加依赖：rand = "0.7"
use std::iter::from_fn;
// 产生1000 个随机数，在[0, 1] 之间均匀分布。
// （这并不是你想在`rand_distr` crate 中找到的分布，
// 但你可以很容易地自己实现它）
let lengths: Vec<f64> =
from_fn(|| Some((random::<f64>() - random::<f64>()).abs()))
.take(1000)
.collect();
```

如果每一个item 都依赖上一个，std::iter::successors 函数是更好的选择。只需要提供一个初始item 和一个函数，这个函数要获取上一个item 并返回下一个item 的Option。如果返回None，那么迭代终止。

from_fn 和successors 都接受FnMut 闭包，因此可以捕获并修改作用域中的变量。

例如，fibonacci 函数使用一个move 闭包来捕获一个变量并使用它作为运行状态：
```rust
fn fibonacci() -> impl Iterator<Item=usize> {
    let mut state = (0, 1);
    std::iter::from_fn(move || {
        state = (state.1, state.0 + state.1);
        Some(state.0)
    })
}
assert_eq!(fibonacci().take(8).collect::<Vec<_>>(), vec![1, 1, 2, 3, 5, 8, 13, 21]);
```

### （四）drain 方法

drain方法主要用于从集合（如Vec、HashMap等）中移除元素，并返回一个迭代器，这个迭代器可以用于遍历被移除的元素。这个方法在需要同时移除和处理集合中的元素时非常有用。
```rust
use std::iter::FromIterator;
let mut outer = "Earth".to_string();
let inner = String::from_iter(outer.drain(1..4));
assert_eq!(outer, "Eh");
assert_eq!(inner, "art");
```

如果你确实要消耗整个序列，使用整个范围.. 作为参数

## 三 迭代器适配器：代码的 “魔法棒”

Iterator trait 还提供了广泛的适配器方法(adapter method)，或者简称为适配器(adapter)，它们消耗一个迭代器然后构建一个新的迭代器。以下方法包括：截断、跳过、组合、反向、连接、重复，等等。
```rust
// 类似javascript中的map，但返回的是新的迭代器
fn map<B, F>(self, f: F) -> impl Iterator<Item=B>
where Self: Sized, F: FnMut(Self::Item) -> B;
// 过滤
fn filter<P>(self, predicate: P) -> impl Iterator<Item=Self::Item>
where Self: Sized, P: FnMut(&Self::Item) -> bool;

// 兼具filter和map
fn filter_map<B, F>(self, f: F) -> impl Iterator<Item=B>
where Self: Sized, F: FnMut(Self::Item) -> Option<B>;
// 兼具flaten和map
fn flat_map<U, F>(self, f: F) -> impl Iterator<Item=U::Item>
where F: FnMut(Self::Item) -> U, U: IntoIterator;
// 拍平
fn flatten(self) -> impl Iterator<Item=Self::Item::Item>
where Self::Item: IntoIterator;

//Iterator trait 的take 和take_while 适配器，可以在迭代了一定的次数或者当一个闭包
决定截断时停止迭代
fn take(self, n: usize) -> impl Iterator<Item=Self::Item>
where Self: Sized;
fn take_while<P>(self, predicate: P) -> impl Iterator<Item=Self::Item>
where Self: Sized, P: FnMut(&Self::Item) -> bool;

// Iterator trait 的skip 和skip_while 方法是take 和take_while 的补充：它们丢弃迭代起
始的一定数量的item，或者直到一个闭包找到一个可接受的item 时，按原样传递这个和剩余
的item。
fn skip(self, n: usize) -> impl Iterator<Item=Self::Item>
where Self: Sized;
fn skip_while<P>(self, predicate: P) -> impl Iterator<Item=Self::Item>
where Self: Sized, P: FnMut(&Self::Item) -> bool;

// peekable 迭代器可以查看下一个将被产生的item，而不实际消耗它。
fn peekable(self) -> std::iter::Peekable<Self>
where Self: Sized;

// fuse 适配器接受任何迭代器，并产生一个保证第一次返回None之后一直返回None的迭代器
fn fuse(self) -> std::iter::Fuse<Self>
where Self: Sized;

// 可以从序列的任意一端产生item的迭代器，可以通过使用rev 适配器来反转这样的迭代器
fn rev(self) -> impl Iterator<Item=Self>
where Self: Sized + DoubleEndedIterator;

// inspect 适配器用在调试迭代器适配器的流水线中，但通常不用在生产代码中
pub fn inspect<F>(self, f: F) -> Inspect<Self, F>
where Self: Sized, F: FnMut(&Self::Item);

// chain 迭代器适配器将一个迭代器附加到另一个之后
fn chain<U>(self, other: U) -> impl Iterator<Item=Self::Item>
where Self: Sized, U: IntoIterator<Item=Self::Item>;

// enumerate 适配器把一个索引附加到序列中，返回一个产生(0, A), (1, B), (2, C), ... 的迭代器
fn enumerate(self) -> Enumerate<Self>
where Self: Sized;

// zip 适配器将两个迭代器组合成单个迭代器，它一次产生一个pair；当两个底层迭代器中有任何
一个结束时，zip 迭代器也会结束。
fn zip<U>(self, other: U) -> Zip<Self, U::IntoIter>
where Self: Sized, U: IntoIterator;

// by_ref 方法借用一个迭代器的可变引用，当消耗完这些适配器的item 之后，借用结束；还可以重新访问原本的迭代器
fn by_ref(&mut self) -> &mut Self
where Self: Sized;
    
// cloned 适配器，类似于iter.map(|item| item.clone())；被引用的类型必须实现Clone
fn cloned<'a, T>(self) -> Cloned<Self>
where T: 'a, Self: Sized + Iterator<Item = &'a T>, T: Clone;
    
// copied 适配器也是相同的思路，但更加严格：被引用的类型必须实现Copy。iter.copied() 调用类似于iter.map(|r| *r)。
fn copied<'a, T>(self) -> Copied<Self>
where T: 'a, Self: Sized + Iterator<Item = &'a T>, T: Copy;
    
// cycle 适配器返回一个无限重复的迭代器。底层迭代器必须实现std::clone::Clone，
// 这样cycle 才可以保存它的初始状态，然后在每一次循环开始时重用它
fn cycle(self) -> Cycle<Self>
where Self: Sized + Clone;
```


## 四 消耗迭代器：获取最终结果

创建迭代器及用适配器将它们包装成新的迭代器之后，就是消耗它们来结束这个过程。

可以使用for 循环来消耗一个迭代器，或者显式地调用next，也可以使用消耗迭代器来进行处理。
```rust
// 简单的累计：count(求数量), sum（求和）, product（求乘）
pub fn count(self) -> usize
where Self: Sized
    
pub fn sum<S>(self) -> S
where Self: Sized, S: Sum<Self::Item>
    
pub fn product<P>(self) -> P
where Self: Sized, P: Product<Self::Item>
    
// Item 的类型必须实现了std::cmp::Ord
pub fn max(self) -> Option<Self::Item>
where Self: Sized, Self::Item: Ord

pub fn min(self) -> Option<Self::Item>
where Self: Sized, Self::Item: Ord

// 通过提供比较函数来判断大小
pub fn max_by<F>(self, compare: F) -> Option<Self::Item>
where Self: Sized, F: FnMut(&Self::Item, &Self::Item) -> Ordering

pub fn min_by<F>(self, compare: F) -> Option<Self::Item>
where Self: Sized, F: FnMut(&Self::Item, &Self::Item) -> Ordering

// max_by_key 和min_by_key 方法通过比较对item 调用闭包返回的值来分别返
回最大值和最小值。
pub fn max_by_key<B, F>(self, f: F) -> Option<Self::Item>
where B: Ord, Self: Sized, F: FnMut(&Self::Item) -> B
    
pub fn min_by_key<B, F>(self, f: F) -> Option<Self::Item>
where B: Ord, Self: Sized, F: FnMut(&Self::Item) -> B,


// 迭代器提供eq 和ne 方法进行相等性比较，以及lt, le, gt, ge 方法用于顺序性比较
pub fn eq<I>(self, other: I) -> bool
where I: IntoIterator, Self::Item: PartialEq<I::Item>, Self: Sized
    
pub fn ne<I>(self, other: I) -> bool
where I: IntoIterator, Self::Item: PartialEq<I::Item>, Self: Sized

pub fn lt<I>(self, other: I) -> bool
where I: IntoIterator, Self::Item: PartialOrd<I::Item>, Self: Sized;
    
 pub fn le<I>(self, other: I) -> bool
where I: IntoIterator, Self::Item: PartialOrd<I::Item>, Self: Sized;
    
pub fn ge<I>(self, other: I) -> bool
where I: IntoIterator, Self::Item: PartialOrd<I::Item>, Self: Sized;

pub fn gt<I>(self, other: I) -> bool
where I: IntoIterator, Self::Item: PartialOrd<I::Item>, Self: Sized;   
          
// any 和all 方法，分别当任意item 使闭包返回true 和所有item 都使闭包返回true 时返回true                                                                        
 pub fn any<F>(&mut self, f: F) -> bool
where Self: Sized, F: FnMut(Self::Item) -> bool;              
                
pub fn all<F>(&mut self, f: F) -> bool
where Self: Sized, F: FnMut(Self::Item) -> bool;                
                
// position 方法返回第一个使闭包返回true 的item 的索引的Option              
pub fn position<P>(&mut self, predicate: P) -> Option<usize>
where Self: Sized, P: FnMut(Self::Item) -> bool;               
// rposition 方法要求一个可逆迭代器及一个exact-size 迭代器
// 这样才可以返回一个和position 一样的索引
pub fn rposition<P>(&mut self, predicate: P) -> Option<usize>
where P: FnMut(Self::Item) -> bool, Self: Sized + ExactSizeIterator + DoubleEndedIterator;
     
// 用于进行累计所有item 的计算                           
pub fn fold<B, F>(self, init: B, mut f: F) -> B
where Self: Sized, F: FnMut(B, Self::Item) -> B;

// rfold 方法和fold 方法相同，只是需要双端迭代器
pub fn rfold<B, F>(self, init: B, mut f: F) -> B
where Self: Sized, F: FnMut(B, Self::Item) -> B, Self: Iterator;               
                
// try_fold 方法和fold 基本一样，但迭代过程可以提前退出，不需要消耗迭代器里的所有值           
pub fn try_fold<B, F, R>(&mut self, init: B, mut f: F) -> R
where Self: Sized, F: FnMut(B, Self::Item) -> R, R: Try<Output = B>;                
                
pub fn try_rfold<B, F, R>(&mut self, init: B, mut f: F) -> R
where Self: Sized, F: FnMut(B, Self::Item) -> R, R: Try<Output = B>, Self: Iterator;              
                
// nth 方法接受一个索引n，跳过迭代器中接下来n个item，然后返回下一个item
 pub fn nth(&mut self, n: usize) -> Option<Self::Item>
 // nth_back 方法类似，只是从双端迭代器的尾部往前移动。               
pub fn nth_back(&mut self, n: usize) -> Option<Self::Item>
where Self: Iterator;  
              
// last 方法返回迭代器产生的最后一个item，或者如果迭代器为空时返回None
// 会消耗迭代器的所有item，即使迭代器可逆
fn last(self) -> Option<Self::Item>;

// find 方法从迭代器查找item，返回第一个使给定闭包返回true 的item
pub fn find<P>(&mut self, predicate: P) -> Option<Self::Item>
where Self: Sized, P: FnMut(&Self::Item) -> bool;
    
// rfind 方法类似，只是需要双端迭代器并且从最后往前搜索
pub fn rfind<P>(&mut self, predicate: P) -> Option<Self::Item>
where Self: Sized, P: FnMut(&Self::Item) -> bool, Self: Iterator;
    
    
// 这类似于find，区别在于它接受的闭包返回一个Option。find_map 返回第一个是Some 的Option
pub fn find_map<B, F>(&mut self, f: F) -> Option<B>
where Self: Sized, F: FnMut(Self::Item) -> Option<B>;
    
    
// 用于构建标准库中的任意集合，只要迭代器产生合适的item 类型
// 需要B实现FromIterator
pub fn collect<B>(self) -> B
where B: FromIterator<Self::Item>, Self: Sized;
    
// 如果一个类型实现了std::iter::Extend trait，
// 那么它的extend 方法可以把一个可迭代对象的item 添加到集合
trait Extend<A> {
    fn extend<T>(&mut self, iter: T)
    where T: IntoIterator<Item=A>;
}
// 如下extend代码
let mut v: Vec<i32> = (0..5).map(|i| 1 << i).collect();
v.extend(&[31, 57, 99, 163]);
assert_eq!(v, &[1, 2, 4, 8, 16, 31, 57, 99, 163]);

// 和std::iter::FromIterator 非常相似：
// 不过FromIterator创建一个新集合，而Extend 扩展一个已有集合。
// 实际上标准库中的好几个类型的FromIterator 
// 的实现都是简单地创建一个新的空集合，然后调用extend 来填充它

impl<T> FromIterator<T> for LinkedList<T> {
    fn from_iter<I: IntoIter<Item = T>>(iter: I) -> Self {
        let mut list = Self::new();
        list.extend(iter);
        list
    }
}

// partition 方法将一个迭代器的item 分成两个集合，
// 使用一个闭包来决定每个item 属于哪个集合
fn partition<B, F>(self, f: F) -> (B, B)
where Self: Sized, B: Default + Extend<Self::Item>, F: FnMut(&Self::Item) -> bool;

pub fn for_each<F>(self, f: F)
where Self: Sized, F: FnMut(Self::Item);
    
// 闭包需要容错或者提前退出，可以使用try_for_each
pub fn try_for_each<F, R>(&mut self, f: F) -> R
where Self: Sized, F: FnMut(Self::Item) -> R, R: Try<Output = ()>;
```


## 实战：实现一个迭代器

可以为自己的类型实现IntoIterator 和Iterator trait，内置的所有适配器和消耗器，及其他可以和标准库迭代器接口协同工作的库和crate 都可以使用

### （一） 简单的范围类型的迭代器

1、定义一个I32Range自定义类型：
```rust
struct I32Range {
    start: i32,
    end: i32
}
```

2、实现Iterator
```rust
impl Iterator for I32Range {
    type Item = i32;
    fn next(&mut self) -> Option<i32> {
        if self.start >= self.end {
            return None;
        }
        let result = Some(self.start);
        self.start += 1;
        result
    }
}
```


3、使用for循环进行遍历

for 循环会使用IntoIterator::into_iter 来把操作数转换成迭代器。

因为标准库为每一个实现Iterator 的类型自动提供了IntoIterator 实现。
```rust
let mut pi = 0.0;
let mut numerator = 1.0;
for k in (I32Range { start: 0, end: 14 }) {
    pi += numerator / (2*k + 1) as f64;
    numerator /= -3.0;
}
pi *= f64::sqrt(12.0);
// IEEE 754 标准指定了精确的结果。
assert_eq!(pi as f32, std::f32::consts::PI);
```

I32Range 是一种特殊情况，它的可迭代对象和迭代器恰好是同一种类型。

### （二）实现二叉树迭代器

1、定义二叉树
```rust
enum BinaryTree<T> {
    Empty,
    NonEmpty(Box<TreeNode<T>>)
}
struct TreeNode<T> {
    element: T,
    left: BinaryTree<T>,
    right: BinaryTree<T>
}
```


2、定义迭代器类型

当为BinaryTree 实现Iterator 时，每一次next 调用都必须产生一个值并且返回。

为了追踪要产生的树节点，迭代器必须保持自己的栈。
```rust
use self::BinaryTree::*;
// 中序遍历`BinaryTree`时的状态
struct TreeIter<'a, T> {
    // 一个树节点的引用的栈。因为我们要使用`Vec`的
    // `push`和`pop`方法，栈的顶部是vector 的尾部。
    //
    // 迭代器下一个要访问的节点在栈顶，
    // 那些还没访问的祖先节点在它下面。如果栈为空，
    // 就代表迭代结束了。
    unvisited: Vec<&'a TreeNode<T>>
}
```


3、实现迭代器的节点生成函数

创建一个新的TreeIter 时，初始状态是树中最左边的节点。

根据unvisited 栈的规则，栈顶应该是那个叶节点，再往下就是它的祖先节点：树中左侧边缘上的节点。

可以从根节点访问左侧边缘一直到叶节点，把遇到的所有节点都入栈，来初始化unvisited。
```rust
impl<'a, T: 'a> TreeIter<'a, T> {
    fn push_left_edge(&mut self, mut tree: &'a BinaryTree<T>) {
        while let NonEmpty(ref node) = *tree {
            self.unvisited.push(node);
            tree = &node.left;
        }
    }
}
```


4、为BinaryTree实现iter方法

给BinaryTree 一个iter 方法来返回一个迭代树的迭代器
```rust
impl<T> BinaryTree<T> {
    fn iter(&self) -> TreeIter<T> {
        let mut iter = TreeIter { unvisited: Vec::new() };
        iter.push_left_edge(self);
        iter
    }
}
```


5、为BinryTree实现IntoIterator

遵循标准库的实践，为二叉树的引用实现IntoIterator，调用BinaryTree::iter方法。

IntoIter 定义将TreeIter 设置为&BinaryTree 的迭代器类型。
```rust

impl<'a, T: 'a> IntoIterator for &'a BinaryTree<T> {
    type Item = &'a T;
    type IntoIter = TreeIter<'a, T>;
    fn into_iter(self) -> Self::IntoIter {
        self.iter()
    }
}
```


6、为TreeIter类型实现Iterator

TreeIter类型需要实现Iterator的next方法。
```rust
impl<'a, T> Iterator for TreeIter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<&'a T> {
        // 找到这一次迭代要产生的节点，
        // 或者结束迭代。（如果结果是`None`就通过
        // `?`运算符立即返回。）
        let node = self.unvisited.pop()?;
        // 在`node`之后，下一个产生的应该是`node`的右子树
        // 中的最左侧的节点，因此添加这条线上的节点。我们的辅助函数
        // 恰好就是我们需要的。
        self.push_left_edge(&node.right);
        // 产生这个节点的值的引用。
        Some(&node.element);
    }
}
```


7、for循环遍历

实现了IntoIterator 和Iterator 后，就可以使用一个for 循环来迭代BinaryTree 的引用。
```rust
// 建造一棵小树。
let mut tree = BinaryTree::Empty;
tree.add("Jaeger");
tree.add("robot");
tree.add("droid");
tree.add("mecha");

let mut v = Vec::new();
for kind in &tree {
    v.push(*kind);
}
assert_eq!(v, ["droid", "Jaeger", "mecha", "robot"]);
```


8、其他迭代方式

所有常用的迭代器适配器和消耗器都可以正常使用。
```rust
assert_eq!(tree.iter()
.map(|name| format!("mega-{}", name))
.collect::<Vec<_>>(),
vec!["mega-droid", "mega-jaeger", "mega-mecha", "mega-robot"]);
```
