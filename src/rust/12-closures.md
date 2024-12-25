![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 闭包：高效编程的秘密武器


## 一、引言

闭包（Closures）为开发者提供了一种灵活且高效的编程方式。闭包在 Rust 中被广泛应用于众多领域，如迭代器操作、异步编程、事件处理以及回调函数等。

它能够捕获其所在环境中的变量，使得开发者可以方便地在不同的上下文中使用这些变量，从而实现更加简洁、灵活和可维护的代码。

## 二、闭包基础

在 Rust 中，闭包（Closures）是一种特殊的函数，它能够捕获其所在环境中的变量，形成一个封闭的上下文。

与普通函数不同的是，闭包可以 “记住” 它被创建时的环境，并且在后续的调用中继续使用这些环境中的变量。
```rust
fn main() { 
    let x = 5; 
    let closure = || println!("The value of x is: {}", x); 
    closure(); 
}
```
在上述代码中，closure 就是一个闭包，它捕获了外部变量 x。当闭包被调用时，它能够打印出 x 的值，即使在闭包定义之后，x 的作用域已经结束。

普通函数在定义时，其参数和局部变量在函数执行完毕后就会被销毁。而闭包则可以将其捕获的变量保存下来，延长了这些变量的生命周期，直到闭包本身被销毁。这使得闭包在处理一些需要保留状态的场景时非常有用，比如在迭代器中使用闭包来保存迭代的状态。

## 三、捕获变量

### （一）引用捕获

当闭包只需要读取外部变量而不需要修改它时，会通过引用捕获变量。
```rust
fn main() { 
    let x = 5; 
    let closure = || println!("The value of x is: {}", x); 
    closure(); 
}
```
在这个例子中，closure 闭包通过引用捕获了变量 x。因为闭包内部只是打印 x 的值，不需要对其进行修改。

### （二）可变引用捕获

如果闭包需要修改外部变量，则通过可变引用捕获。例如：
```rust
fn main() { 
    let mut x = 5; 
    let mut closure = || { 
        x += 1; 
        println!("The value of x is: {}", x); 
    }; 
    closure(); 
    closure(); 
}
```
这里的 closure 闭包通过可变引用捕获了变量 x，在闭包体内对 x 进行了自增操作，并打印出每次修改后的 x 值。由于闭包持有对 x 的可变引用，在闭包调用期间，其他代码不能同时获取对 x 的可变引用，以避免数据竞争。

### （三）移动捕获

使用 move 关键字可以将外部变量的所有权转移到闭包中，这就是移动捕获。
```rust
fn main() { 
    let s = String::from("Hello"); 
    let closure = move || println!("The string is: {}", s); 
    closure(); 
    // 下面这行代码会报错，因为 s 的所有权已经被移动到闭包中 
    // println!("{}", s); 
}
```
在这个例子中，s 是一个 String 类型的变量，它的所有权被移动到了 closure 闭包中。这意味着在闭包之后，不能再使用 s。

移动捕获通常用于将变量的所有权转移到闭包中，以便在闭包的生命周期内独占使用该变量，或者将闭包传递到其他线程中时，确保变量的所有权正确转移。

## 四、闭包类型

### （一）FnOnce

FnOnce 类型的闭包会获取其环境中变量的所有权，并且只能被调用一次。

这是因为在闭包调用时，变量的所有权被转移到了闭包内部，闭包执行完毕后，该变量的生命周期就结束了。
```rust
fn main() { 
    let x = vec![1, 2, 3]; 
    let closure = || { 
        let y = x; 
        println!("{:?}", y) 
    }; 
    closure(); 
    // 下面这行代码会报错，因为 x 的所有权已经被移动到闭包中，闭包调用后 x 已被释放 
    // println!("{:?}", x); 
}
```
在这个例子中，closure 闭包通过 将x赋值y，获取了 x 的所有权，所以只能调用一次 closure。当再次尝试访问 x 时，编译器会报错，提示 x 已经被移动且不再可用。

FnOnce 闭包适用于那些只需要在闭包中使用一次变量，并且之后不再需要该变量的场景，比如在某些初始化操作中，将变量的所有权转移到闭包内进行一次性的处理。

### （二）FnMut

FnMut 闭包可以可变地借用其环境中的变量，这意味着它能够修改被借用的变量。这种闭包可以被多次调用，每次调用都可以对变量进行修改。
```rust
fn main() { l
    et mut x = 5; 
    let mut closure = || { 
        x += 1; 
        println!("The value of x is: {}", x); 
    }; 
    closure(); 
    closure(); 
}
```
在上述代码中，closure 是一个 FnMut 闭包，它可变地借用了变量 x。

每次调用 closure 时，都会将 x 的值增加 1 并打印出来。

由于闭包是可变借用 x，所以在闭包的多次调用之间，x 的值会持续发生变化。FnMut 闭包常用于需要在多个操作步骤中修改某个变量状态的情况，如在一个循环中不断更新某个计数器或迭代器的内部状态。

### （三）Fn

Fn 闭包不可变地借用其环境中的变量，它也可以被多次调用，但不能修改被借用的变量。
```rust
fn main() { 
    let x = 5; 
    let closure = || println!("The value of x is: {}", x); 
    closure(); 
    closure(); 
}
```
这里的 closure 闭包是 Fn 类型，它只是读取并打印变量 x 的值，而不会对 x 进行任何修改。

在多次调用 closure 时，x 的值始终保持不变。Fn 闭包在函数式编程中非常常用，例如在对集合进行遍历操作时，使用 Fn 闭包来定义每个元素的处理逻辑，而不会改变集合本身的结构和元素。

需要注意的是，在 Rust 中，闭包的类型是根据其对环境变量的使用方式自动推断的。一个闭包可能同时实现了 Fn、FnMut 和 FnOnce 中的多种 trait，但具体调用时的行为取决于上下文对闭包类型的要求。

它们之前的关系是：Fn是FnMut的一个子集，而FnMut 又是FnOnce的一个子集。

注意⚠️：函数和闭包在类型上的差别：
```rust
fn() -> bool // fn 类型（只限函数） 
Fn() -> bool // Fn trait（函数和闭包）
```
## 五、闭包的性能

### （一）与其他语言的差别

在大多数语言中，闭包都在堆上分配、动态分发、被垃圾收集器回收。

因此创建、调用、回收每个闭包都需要消耗额外的CPU 时间，而且，闭包常常会排除内联(inline)。

但是Rust的闭包没有垃圾回收，并且除非是放在Box、Vec 或其他容器中，否则它们不会在堆上分配。

### （一）优化策略

在 Rust 中，编译器会对闭包进行多种优化，以提升其性能表现。

当闭包只在一个地方被调用时，编译器可能会内联这个闭包，直接将闭包的代码展开到调用处，从而避免了函数调用的开销。
```rust
fn main() { 
    let x = 5; 
    let closure = || x + 1; 
    let result = closure(); 
    println!("{}", result); 
}
```
另外，根据闭包捕获变量的方式，编译器也会优化内存使用。

如果是通过引用捕获变量，只要保证引用的合法性（遵循借用规则等），就可以高效地利用已有的变量内存空间，而无需额外为变量进行重复的内存分配。
```rust
fn main() { 
    let mut data = vec![1, 2, 3]; 
    let closure = || data.len(); 
    println!("Length of data: {}", closure()); 
}
```
###

## 六、闭包的Copy 和Clone

闭包被表示为包含它们捕获的值（move 闭包）或者引用（non-move 闭包）的结构体。

闭包的Copy 和Clone 的规则就类似于普通结构体的Copy 和Clone 的规则。

### （一）Fn

一个没有可变变量的non-move 的闭包只有共享引用，共享引用是Clone 和Copy，所以这种闭包也是Clone 和Copy。
```rust
let y = 10; 
let add_y = |x| x + y; 
let copy_of_add_y = add_y; // 这个闭包是`Copy`
assert_eq!(add_y(copy_of_add_y(22)), 42); // 可以使用这两个
```
### （二）FnMut

一个有可变值的non-move 闭包在内部的表示中包含可变引用。可变引用既不是Clone 也不是Copy，因此这样的一个闭包既不是Copy 也不是Clone。
```rust
let mut x = 0; 
let mut add_to_x = |n| { 
    x += n; 
    x 
}; 
let copy_of_add_to_x = add_to_x; // 移动，而不是拷贝 
assert_eq!(add_to_x(copy_of_add_to_x(1)), 2); // 错误：使用了被移动的值
```
### （三）FnOnce

对于move 闭包来说，规则变得更简单了。

如果一个move 闭包捕获的所有值都是Copy，那么它也是Copy。如果它捕获的所有值都是Clone，那么它也是Clone。
```rust
let mut greeting = String::from("Hello, "); 
let greet = move |name| { 
    greeting.push_str(name); 
    println!("{}", greeting); 
}; 
greet.clone()("Alfred"); // Hello, Alfred 
greet.clone()("Bruce"); // Hello, Bruce
```
当greeting 在greet 中被用到时，它会被移动进表示greet 的内部结构体里，因为它是move 闭包。

当克隆greet 时，里面的所有内容都会被克隆。

这里有两个greeting 的拷贝，当调用克隆的greet 时它们会被独立地修改。

它本身用处不大，但当你需要把同样的闭包传递给不止一个函数时，它会很有用。