![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 集合：编程中的得力助手

## 一、Rust 集合
Rust 中的集合(Collection)，大多存储在堆上，大小可以在运行时动态改变，不像数组那样需要在编译时就确定固定的大小。
它们都遵循 Rust 严格的所有权和借用规则，确保内存安全，无需担心常见的内存错误，如悬空指针、缓冲区溢出等。
这些集合涵盖了从简单的线性存储，到键值对映射，再到特殊用途的集合。
| 集合                              | 说明                                                                                   |
| :-------------------------------- | :------------------------------------------------------------------------------------- |
| Vec\<T\>                          | 一个可增长的、在堆上分配的、T 类型值的数组                                             |
| VecDeque\<T>                      | 双端队列(可增长环形缓冲区), 先进先出队列。支持高效地在首部和尾部添加或移除元素         |
| LinkedList\<T>                    | 双向链表                                                                               |
| BinaryHeap\<T> where T: Ord       | 优先队列，大顶堆, BinaryHeap中的值按照一定结构组织，因此总是可以高效地找到和移除最大值 |
| HashMap\<K, V> where K: Eq + Hash | 键值哈希表, 一个键值对的表,通过键查找值很快速                                          |
| BTreeMap\<K, V> where K: Ord      | 有序键值表, 按键的顺序保持条目有序                                                     |
| HashSet\<T> where T: Eq + Hash    | 基于哈希的无序集合                                                                     |
| BTreeSet\<T> where T: Ord         | 有序集合                                                                               |
## 二、动态数组 Vector
Vector，通常简称为 Vec，是 Rust 中最常用的集合之一。
Vector在内存中有三个字段：长度、容量、和一个指向堆上分配的缓冲区的指针。
### 1、Vec\<T> 的定义：
```rust
pub struct Vec<T, A = Global>
where
    A: Allocator,
{
    buf: RawVec<T, A>,
    len: usize,
}
```
### 2、Vec\<T> 的创建：
```rust
// 创建一个空的 Vector 可以用 Vec::new ()
let mut numbers: Vec<i32> = Vec::new();
numbers.push(1);
numbers.push(2);
// 也可以使用 vec! 宏来快速创建并初始化 Vector，像这样：
let fruits = vec!["apple", "banana", "cherry"];
let words = vec!["step", "on", "no", "pets"];
let mut buffer = vec![0u8; 1024]; // 1024 个0 字节

let mut set = HashSet::new();
set.insert("a".to_string());
set.insert("b".to_string());
set.insert("c".to_string());
//Vec 实现了std::iter::FromIterator， 将一个其他集合转换成vector
let my_vec = set.into_iter().collect::<Vec<String>>();

```

### 3、访问元素
访问元素，有两种方式。
一种是使用索引，要小心越界问题。
Rust 不会自动检查，越界会导致程序 panic, 并且vector 的长度和索引都是usize 类型：
```rust
// 获取一个元素的引用
let first_line = &lines[0];
// 获取一个元素的拷贝
let fifth_number = numbers[4]; 
let second_number = lines[1].clone(); // 需要Copy
// 需要Clone
// 获取一个切片的引用
let my_ref = &buffer[4..12];
// 获取一个切片的拷贝
let my_copy = buffer[4..12].to_vec(); // 需要Clone
```
另一种更安全的方式是使用 get 方法，它返回一个 Option<&T>，如果索引无效则返回 None，避免了 panic。
```rust
let slice = [0, 1, 2, 3];
assert_eq!(slice.get(2), Some(&2));
assert_eq!(slice.get(4), None);

// 返回slice 的第一个元素的引用。返回类型是Option<&T>
if let Some(item) = slice.first() {
    println!("We got one! {}", item);
}
// 返回slice 的最后一个元素的引用。返回类型是Option<&T>
if let Some(item) = slice.last() {
    println!("We got one! {}", item);
}

// slice.first_mut(), slice.last_mut(), slice.get_mut(index), 可变引用 
let last = slice.last_mut().unwrap(); 
assert_eq!(*last, 3);
*last = 100;

```
### 4、遍历元素
vector 和切片可以以值或者以引用迭代，遵循IntoIterator 实现中介绍的模式。
数组、切片和vector 还有 .iter() 和 .iter_mut() 方法创建产生元素的引用的迭代器。
```rust
let slice = [0, 1, 2, 3];
let slice2 = slice.map(|x| {
    x * 2
});
let slice3 = slice.iter().map(|x| {
    x * 2
}).collect::<Vec<i32>>();
```
## 三、 双端队列 VecDeque
VecDeque（发音为 “vector deck”）是一个双端队列，它支持在头部和尾部高效地添加和移除元素.

### 1、VecDeque\<T> 的定义：
```rust
pub struct VecDeque<T, A = Global>
where
    A: Allocator,
{
    head: usize,
    len: usize,
    buf: RawVec<T, A>,
}
```
很多Vec 的方法在VecDeque 中都有实现：.len(), .is_empty(), .insert(index, value), .remove(index), .extend(iterable)，等等
### 2、VecDeque\<T> 的使用：
```rust
// 创建一个空的 VecDeque
let mut deque: VecDeque<i32> = VecDeque::new();
// 在头部插入元素
deque.push_front(1);
// 在尾部插入元素
deque.push_back(2);
// 在头部弹出元素
if let Some(front) = deque.pop_front() {
    println!("Popped from front: {}", front);
}
// 在尾部弹出元素
if let Some(back) = deque.pop_back() {
    println!("Popped from back: {}", back);
}
// 遍历元素
for element in &deque {
    println!("{}", element);
}
// 类似于vec.first() 和vec.last()
let r_back = deque.back();
let r_front = deque.front();
```
### 3、VecDeque\<T> 的实现
VecDeque 的实现是一个**环形缓冲区**。
类似于Vec，它有一个堆上分配的缓冲区用来存储元素。
不同的是，数据并不总是从区域的起点开始存储，并且可以在结尾处“回环”。
下图中的队列元素，按顺序分别是['A', 'B', 'C', 'D', 'E']。
VecDeque 的私有的字段标记图中的start 和stop，用来记录缓冲区中数据的起点和终点。
向队列首部或者尾部添加值，意味着要占用一个未使用的位置，如果需要的话可能会回环或者分配更大的内存块。


![VecDeque.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/48c346e1b3004c78a211c904dd234e3d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1736682955&x-orig-sign=S05ywDu1Owp2NhaVIS9LcchoiM4%3D)


因为双端队列在内存中并不是连续存储元素，因此它不能继承切片的方法。
但VecDeque 提供了一个方法来修复它：
```rust
deque.make_contiguous()
```

## 四、 优先队列 BinaryHeap
BinaryHeap 是一个优先队列
默认实现为最大堆（可以通过自定义实现最小堆），它保证每次弹出的元素都是堆中的最大（或最小）元素。
插入和弹出操作的时间复杂度为 O (log n)。
### 1、BinaryHeap\<T> 的定义：
```rust
pub struct BinaryHeap<T, A = Global>
where
    A: Allocator,
{
    data: Vec<T, A>,
}
```
### 2、BinaryHeap\<T> 的使用：
```rust
// 创建一个空的 BinaryHeap
let mut heap: BinaryHeap<i32> = BinaryHeap::new();
// 插入元素
heap.push(1);
heap.push(2);
heap.push(3);
// 弹出最大元素
if let Some(max) = heap.pop() {
    println!("Popped max: {}", max);
}// 引用最大元素
if let Some(max) = heap.peek() {
    println!("Popped max: {}", max);
}
// 遍历元素
for element in &heap {
    println!("{}", element);
}

let mut heap = BinaryHeap::from(vec![2, 3, 8, 6, 9, 5, 4]);
// 值9 在堆的顶部：
assert_eq!(heap.peek(), Some(&9));
assert_eq!(heap.pop(), Some(9));
// 移除9 也会重新排布其他元素，把8 移动到头部，等等：
assert_eq!(heap.pop(), Some(8));
assert_eq!(heap.pop(), Some(6));
assert_eq!(heap.pop(), Some(5));
```
## 五、 HashMap<K, V> 和 BTreeMap<K, V>
map(映射) 是键值对（称为条目(entry)）的集合。
任何两个条目的键都不同，所有的条目按照一定结构组织。
Rust 提供两者两种map 类型：HashMap<K, V> 和BTreeMap<K, V>。
这两种类型共享了很多相同的方法；
不同之处在于它们组织条目的方式。
### 1、HashMap<K, V> 与 BTreeMap<K, V> 的定义：
```rust
pub struct HashMap<K, V, S = RandomState> {
    base: {unknown},
}

pub struct BTreeMap<K, V, A = Global>
where
    A: Allocator + Clone,
{
    root: Option<NodeRef<Owned, K, V, LeafOrInternal>>,
    length: usize,
    pub(super) alloc: ManuallyDrop<A>,
    _marker: PhantomData<Box<(K, V), A>>,
}
```
### 2、HashMap<K, V> 与 BTreeMap<K, V> 的使用：
```rust
// 创建一个空的 HashMap
let mut map: HashMap<String, i32> = HashMap::new();
// 插入键值对
map.insert(String::from("Alice"), 30);
map.insert(String::from("Bob"), 25);
// 插入或更新键值对
map.entry(String::from("Alice")).or_insert(35);
// 访问值
if let Some(age) = map.get(&String::from("Alice")) {
    println!("Alice's age is: {}", age);
}
// 遍历键值对
for (key, value) in &map {
    println!("{}: {}", key, value);
}

// 创建一个空的 BTreeMap
let mut map: BTreeMap<String, i32> = BTreeMap::new();
// 插入键值对
map.insert(String::from("Alice"), 30);
map.insert(String::from("Bob"), 25);
// 插入或更新键值对
map.entry(String::from("Alice")).or_insert(35);
// 访问值
if let Some(age) = map.get(&String::from("Alice")) {
    println!("Alice's age is: {}", age);
}
// 遍历键值对
for (key, value) in &map {
    println!("{}: {}", key, value);
}
```

### 3 HashMap<K, V> 与 BTreeMap<K, V> 的内存排布
#### (1) HashMap<K, V> 的内存排布
所有的键、值和缓存的哈希值都被存储在单个堆上分配的表中。
添加条目最终会迫使HashMap 分配更大的表，并把所有数据移动进去。


![HashMap.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/b05a2ac2d9264dcd9e842a11602a835e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1736682997&x-orig-sign=cp5p9CbeMxfLyP8TmDQS6rgolHY%3D)


#### (2) BTreeMap<K, V> 的内存排布
BTreeMap 按照键的顺序在树形结构中存储条目，因此它要求键的类型K 实现了Ord

BTreeMap 把条目存储在节点(node) 中。
一个BTreeMap 中大多数的节点都只包含键值对,而非叶节点。
添加条目通常需要把某个节点的部分向现有条目向右移动，来保持它们有序，有时还需要分配新的节点。

![BTreeMap.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/177d65e7f58346618f6cd3ef3d44be0e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1736683303&x-orig-sign=31VvozJ%2BQlhr3t2on4bWIhnnbNI%3D)

## 六、 HashSet<T> 和 BTreeSet<T>
set 是值的集合，它可以快速地测试元素。
一个set 绝不会包含同一个值的多个拷贝。
HashSet 是一个无序的集合，它只存储不重复的元素，内部基于哈希表实现，保证插入、删除和查询的平均时间复杂度为 O (1)。
BTreeSet 是一个基于二叉搜索树实现的有序集合，它保证元素的插入、删除和查询操作的时间复杂度为 O (log n)，且元素始终有序存储。
### 1、HashSet\<T> 与 BTreeSet\<T> 的定义：

```rust
pub struct HashSet<T, S = RandomState> {
    base: {unknown},
}
pub struct BTreeSet<T, A = Global>
where
    A: Allocator + Clone,
{
    map: BTreeMap<T, SetValZST, A>,
}
```

### 2、HashSet\<T> 与 BTreeSet\<T> 的使用：
```rust
// 创建一个空的 HashSet
let mut set: HashSet<i32> = HashSet::new();
// 插入元素
set.insert(1);
set.insert(2);
// 插入或更新元素
set.entry(1).or_insert(35);
// 测试元素是否存在
if set.contains(&1) {
    println!("1 is in the set");
}
// 遍历元素
for element in &set {
    println!("{}", element);
}
// 创建一个空的 BTreeSet
let mut set: BTreeSet<i32> = BTreeSet::new();
// 插入元素
set.insert(1);
set.insert(2);
// 插入或更新元素
set.entry(1).or_insert(35);
// 测试元素是否存在
if set.contains(&1) {
    println!("1 is in the set");
}
// 遍历元素
for element in &set {
    println!("{}", element);
}
```

## 七、 链表 LinkedList
LinkedList 是一种链式存储的数据结构，每个节点包含数据和指向下一个节点的指针（在双向链表中还有指向前一个节点的指针）。
Rust 标准库中的 LinkedList 提供了丰富的操作方法。
LinkedList 很少使用，并且在大多数情况下都有更好的替代，无论是性能还是接口。
```rust
use std::collections::LinkedList;
let mut list = LinkedList::new();
// 在头部插入元素用 push_front，在尾部插入用 push_back：
list.push_front(1);
list.push_back(2);
// 遍历链表可以使用迭代器，比如：
for value in &list {
    println!("{}", value);
}
// 还可以获取链表的头节点（front）、头节点的可变引用（front_mut），弹出头节点（pop_front），弹出尾节点（pop_back）等操作，满足不同的需求：
if let Some(front) = list.front() {
    println!("Front value: {}", front);
}
if let Some(mut front) = list.front_mut() {
    *front += 10;     // 修改头节点的值
}
if let Some(pop_value) = list.pop_front() {
    println!("Popped value: {}", pop_value);
}
```
## 八、 哈希
Hash 是标准库用于可哈希类型的trait。
HashMap的键和HashSet的元素必须实现Hash 和Eq。
大多数实现了Eq 的内建类型也都实现了Hash。
整数类型、char、String 都是可哈希类型；
当元素是可哈希类型时，元组、数组、切片、vector 也是可哈希类型。
标准库的原则是无论一个值存储到哪里或者如何指向它，它必须有相同的哈希值。
因此，引用和被引用的值有相同的哈希值。
Box和被装箱的值有相同的哈希值。
一个vector和包含它的所有元素的切片&vec[..] 有相同的哈希值。
一个String和一个有相同字符的&str有相同的哈希值。
结构和枚举默认没有实现Hash，但可以派生实现：
```rust
#[derive(Clone, PartialEq, Eq, Hash)]
enum MuseumNumber {
    ...
}
```
只要所有的字段都是可哈希的就可以正常工作。


如果手动为一个类型实现了PartialEq，那也应该手动实现Hash。
```rust
struct Artifact {
    id: MuseumNumber,
    name: String,
    cultures: Vec<Culture>,
    date: RoughTime,
    ...
}
```
有相同ID的两个Artifact被认为相等：
```rust
impl PartialEq for Artifact {
    fn eq(&self, other: &Artifact) -> bool {
        self.id == other.id
    }
}
impl Eq for Artifact {}
```
当比较两个Artifact时只根据ID 进行比较，因此必须用相同的方式哈希它们：
```rust
use std::hash::{Hash, Hasher};
impl Hash for Artifact {
    fn hash<H: Hasher>(&self, hasher: &mut H) {
        // 把哈希操作委托给MuseumNumber
        self.id.hash(hasher);
    }
}
```
这样就可以创建一个Artifact 的HashSet：
```rust
let mut collection = HashSet::<Artifact>::new();
```