![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 引用：强大而安全的数据使用特性


# 一、Rust 引用概述

引用是 Rust 中一种重要的机制，它允许在不获取所有权的情况下访问和操作数据。

Rust 中有不可变引用和可变引用两种类型。

- 不可变引用允许以只读方式访问数据，使用&（读作 ref）符号创建。例如：
```rust
let mut x = 5;
let y = &x;
println!("x: {}", x);
println!("y: {}", y);

```
在这个例子中，y是对x的不可变引用，可以读取x的值，但不能修改它。

- 可变引用允许以读写方式访问和修改数据，使用&mut符号创建。例如：
```rust
let mut x = 5;
let y = &mut x;
*y += 1;
println!("x: {}", x);
println!("y: {}", y);
```

这里，y是对x的可变引用，可以通过解引用操作符*(读作 deref)来修改x的值。

引用在使用时需要遵守一些规则以确保内存安全。

- 同一时间只能存在一个可变引用或多个不可变引用，但不能同时存在可变引用和不可变引用。
- 引用必须始终有效，即被引用的数据不能在引用的生命周期内被销毁。

Rust 的编译器会在编译时静态检查这些规则，并在编译阶段防止出现悬垂引用和数据竞争等错误。

总的来说，Rust 的引用机制实现了灵活的数据共享和临时访问，同时保证了内存安全和避免数据竞争。

# 二、Rust 引用的类型

## （一）不可变引用

不可变引用，也被称为共享引用，允许以只读方式访问数据，使用&符号创建。

在 Rust 中，多个不可变引用可以同时指向同一个数据，这确保了数据的安全性，因为没有任何一个引用可以修改数据。例如：
```rust
let mut x = 5;
let y = &x;
let z = &x;
println!("x: {}", x);
println!("y: {}", y);
println!("z: {}", z);
```
在这个例子中，y和z都是对x的不可变引用，可以读取x的值，但不能修改它。这种特性使得代码更加安全，避免了意外的数据修改。

## （二）可变引用

可变引用可以修改引用数据内容，使用&mut符号创建。

但可变引用有一定限制，同一时间只能有一个可变引用指向同一数据。

rus如果存在可变引用，就不能再有其他可变或不可变引用。例如：
```rust
let mut x = 5;
let y = &mut x;
*y += 1;
println!("x: {}", x);
println!("y: {}", y);
```

这里，y是对x的可变引用，可以通过解引用操作符*来修改x的值。

这个限制是为了防止数据竞争，确保在同一时间只有一个引用可以修改数据。

# 三、Rust 引用的规则与限制

## （一）特定作用域的引用限制

在 Rust 中，特定作用域对引用有着严格的限制。

要么在特定作用域内有且只有一个可变引用，要么只能有多个不可变引用，但不能同时存在可变引用和不可变引用。例如：
```rust
let mut guess = String::from("hellow");//特定作用域拥有且只允许有一个可变引用
let r2 = &mut guess;// 如果此时再创建一个不可变引用或者可变引用都会报错
// let r1 = &guess;
// let r3 = &mut guess;
```

这种限制确保了数据的安全性和一致性，防止出现数据竞争的情况。

如果在特定作用域内同时存在可变引用和不可变引用，那么就可能出现数据不一致的问题，因为可变引用可以修改数据，而不可变引用只能读取数据。

## （二）引用必须有效

引用在特定作用域内必须总是有效，这意味着被引用的数据不能在引用的生命周期内被销毁。例如：
```rust
fn dangle() -> &String {
    let s = String::from("test");
    &s
}
```

这段代码会报错，因为s在函数结束时就被销毁了，而返回的引用指向的内存已经被释放，从而产生了垂悬引用。Rust 的编译器会在编译时检查引用的有效性，确保引用指向的内存在引用的生命周期内是有效的。

如果引用指向的内存被提前释放或者分配给其他持有者，编译器会报错，从而避免了潜在的内存安全问题。

# 四、Rust 引用的案例

## （一）函数调用中的引用

以下是一个在函数调用中使用引用的示例：
```rust
fn main() {
    let s1 = String::from("hello");
    let lens = reference(&s1);
    println!("{} 的长度是 {}", s1, lens);
}
// 引用
fn reference(s: &String) -> usize {
    s.len()
}
```

在这个例子中，通过引用将s1传递给reference函数，这样在函数调用过程中，s1的所有权不会被转移，只是借用了s1的值来计算长度。

## （二）数据修改场景中的可变引用

在需要修改数据的场景中，可以使用可变引用。例如：
```rust
fn main() {
    let mut ss = String::from("hello:");
    reference_active(&mut ss);
    println!("变化之后：{}", ss);
}
// 可变参数引用 借用
fn reference_active(some_string: &mut String) {
    some_string.push_str("world");
}
```



这里，通过可变引用&mut ss将可变的字符串ss传递给reference_active函数，在函数内部使用可变引用修改了字符串的值。

## （三）避免所有权转移的引用

在某些情况下，我们希望在多个地方访问同一个数据而不转移所有权。引用可以实现这个目的。例如：
```rust
fn main() {
    let s = String::from("hello");
    let r1 = &s;
    let r2 = &s;
    println!("r1: {}, r2: {}", r1, r2);
}
```

这里，创建了两个不可变引用r1和r2指向同一个字符串s，在不转移s所有权的情况下，可以在多个地方读取s的值。

## （四）引用在复杂数据结构中的应用

在处理复杂数据结构时，引用也非常有用。

例如，在处理链表或树等数据结构时，可以使用引用在不同的节点之间进行导航和操作，而不需要转移所有权。以链表为例：
```rust
enum ListNode {
    Cons(i32, Box<ListNode>),
    Nil,
}
fn main() {
    let mut list = ListNode::Cons(1, Box::new(ListNode::Cons(2, Box::new(ListNode::Nil))));
    let ref1 = &mut list;
    if let ListNode::Cons(val, next) = ref1 {
        println!("Value: {}", val);
        let ref2 = &mut **next;
        if let ListNode::Cons(next_val, _) = ref2 {
            println!("Next value: {}", next_val);
        }
    }
}

```

在这个例子中，使用可变引用在链表中进行遍历和修改操作，而不需要转移链表节点的所有权。
