![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 进阶修炼：掌握 Unsafe 编程的核心能力

## 一、揭开 Rust 的神秘面纱：深入理解 unsafe 关键字

### 1.1 什么是 unsafe？

`unsafe`关键字就像是一把特殊的钥匙，能够打开那些被 Rust 安全机制所限制的大门，让开发者可以执行一些底层的、不受常规安全检查约束的操作。

Rust 语言以其强大的内存安全性和线程安全性而闻名，它通过所有权系统、借用检查和类型系统等机制，确保了在绝大多数情况下，程序不会出现诸如空指针解引用、数据竞争、内存泄漏等常见的安全问题。

然而，在某些特定的场景下，这些安全检查可能会成为实现特定功能或优化性能的阻碍。例如，与 C 语言库进行交互，或者实现一些底层的系统编程功能时，就需要暂时脱离 Rust 的安全检查，使用`unsafe`关键字来执行这些操作。

### 1.2 unsafe 的三大特性

**无自动检查**：`unsafe`代码块中的代码不会像普通 Rust 代码那样，受到编译器严格的内存安全和数据竞争检查。这意味着在`unsafe`块中，开发者需要自己确保代码的正确性和安全性，否则很容易引入难以调试的错误。

**五大超能力**：`unsafe`关键字赋予了开发者五种特殊的能力，这些能力在普通的 Rust 代码中是被禁止的，包括解引用裸指针、调用`unsafe`函数、访问可变静态变量、实现`unsafe` trait、操作`union`。这些能力为开发者提供了更大的灵活性，但同时也带来了更高的风险。

**显式标记**：为了提醒开发者注意`unsafe`代码的潜在风险，Rust 要求所有的`unsafe`代码必须明确地使用`unsafe`关键字进行标记。这样，在阅读和维护代码时，其他开发者可以很容易地识别出哪些部分的代码是不安全的，从而更加谨慎地对待。

### 1.3 为什么需要 unsafe？

**底层系统编程**：在操作系统开发、设备驱动编写或嵌入式系统编程等领域，常常需要直接操作硬件资源或使用特定的内存布局。这些操作往往涉及到对内存的直接读写、指针运算等底层操作，而 Rust 的安全检查机制无法对这些操作进行有效的验证。

**与其他语言交互**：Rust 作为一门新兴的编程语言，虽然发展迅速，但在实际应用中，仍然不可避免地需要与其他语言编写的代码进行交互，尤其是 C 和 C++ 语言。由于 C 和 C++ 语言本身并不具备 Rust 那样严格的安全检查机制，因此在与它们进行交互时，需要使用`unsafe`代码来桥接不同语言之间的边界。

**性能优化**：在某些性能关键的场景中，Rust 的安全检查机制可能会带来一定的性能开销。为了追求极致的性能，开发者可以使用`unsafe`代码来绕过这些检查，实现更高效的算法和数据结构。

下面通过一个简单的示例，来展示`unsafe`代码的基本用法：

```Rust
fn main() {
   let num = 5;
   // 创建裸指针，这一步是安全的
   let ptr: *const i32 = &num as *const i32;

   // 解引用裸指针，需要在 unsafe 块中进行
   unsafe {
       println!("The value pointed by ptr is: {}", *ptr);
   }
}
```

在这个例子中，首先创建了一个指向`num`的不可变裸指针`ptr`。创建裸指针的过程是安全的，因为它只是获取了一个内存地址，并没有对内存进行读写操作。

然而，当我们试图解引用这个裸指针，读取它所指向的值时，就需要使用`unsafe`块来告诉编译器，我们知道这是一个不安全的操作，并且我们已经确保了指针的有效性。

## 二、掌控危险区域：unsafe 块与 unsafe 函数的对比

在 Rust 的`unsafe`编程领域中，`unsafe`块和`unsafe`函数是两个重要的概念，它们各自有着独特的用途和特点。理解它们之间的区别和适用场景，对于编写安全、高效的`unsafe`代码至关重要。

### 2.1 unsafe 块的使用场景

`unsafe`块是一种将一段代码标记为不安全的方式，它允许在其中执行一些在普通 Rust 代码中被禁止的操作。

`unsafe`块的主要作用是将不安全操作的范围限制在一个局部区域内，从而减少潜在的风险。例如，当我们需要解引用裸指针时，就必须将解引用操作放在`unsafe`块中：

```Rust
fn main() {
   let num = 5;
   let ptr: *const i32 = &num as *const i32;
   unsafe {
       println!("The value pointed by ptr is: {}", *ptr);
   }
}
```

`unsafe`块明确地标识了解引用裸指针这一不安全操作的范围，使得代码的安全性和可读性得到了一定的保障。

同时，由于`unsafe`块只影响其内部的代码，外部的代码仍然受到 Rust 安全机制的保护，这就避免了不安全操作对整个程序的影响。

### 2.2 unsafe 函数的契约设计

`unsafe`函数则是一种更高级的抽象，它将一系列不安全操作封装在一个函数中，并通过函数签名和文档来明确其调用者需要满足的契约。`unsafe`函数的主要目的是为了提供一种更方便、更可维护的方式来管理不安全代码。例如，下面是一个简单的`unsafe`函数示例：

```Rust
unsafe fn read_value(ptr: *const i32) -> i32 {
   *ptr
}

fn main() {
   let num = 5;
   let ptr: *const i32 = &num as *const i32;

   unsafe {
       let value = read_value(ptr);
       println!("The value read from ptr is: {}", value);

   }

}
```

在这个例子中，`read_value`函数被标记为`unsafe`，这意味着调用者在调用该函数时必须确保传入的指针是有效的，否则可能会导致未定义行为。

通过将不安全操作封装在函数中，并在函数签名和文档中明确契约，`unsafe`函数使得代码的安全性和可维护性得到了提升。同时，`unsafe`函数还可以被其他`unsafe`函数或`unsafe`块调用，从而实现更复杂的功能。

### 2.3 两者的核心区别

| 特性   | unsafe 块 | unsafe 函数   |
| ---- | -------- | ----------- |
| 作用范围 | 局部代码块    | 全局函数        |
| 安全责任 | 调用者负责    | 实现者与调用者共同负责 |
| 可组合性 | 适合一次性操作  | 适合封装复杂逻辑    |

从作用范围来看，`unsafe`块只影响其内部的代码，而`unsafe`函数则是一个全局的函数，可以在多个地方被调用。这就使得`unsafe`函数更适合用于封装一些通用的不安全操作，而`unsafe`块则更适合用于处理一些临时性的、局部的不安全操作。

在安全责任方面，`unsafe`块的安全责任主要由调用者承担，调用者需要确保`unsafe`块内的操作是安全的；

而`unsafe`函数的安全责任则由实现者和调用者共同承担，实现者需要在函数内部确保操作的正确性，并通过文档明确调用者需要满足的条件，调用者则需要在调用时确保满足这些条件。

最后，从可组合性来看，`unsafe`块通常用于一次性的不安全操作，它的使用相对较为灵活，但不太适合用于封装复杂的逻辑；

而`unsafe`函数则更适合用于封装复杂的不安全逻辑，它可以被多个地方调用，并且可以通过函数参数和返回值来实现更灵活的功能组合。

## 三、构建不安全接口：深入剖析 unsafe trait

### 3.1 定义与实现

在 Rust 的类型系统中，`unsafe trait` 是一种特殊的 `trait`，它允许实现者执行一些编译器无法完全验证安全性的操作。当 `trait` 中的某些方法涉及到裸指针操作、底层内存管理或其他可能破坏内存安全的行为时，这个 `trait` 就应该被定义为 `unsafe`。

定义一个 `unsafe trait` 非常简单，只需在 `trait` 关键字前加上 `unsafe` 关键字即可。例如，我们定义一个 `MyUnsafeTrait`：

```Rust
unsafe trait MyUnsafeTrait {
   unsafe fn do_something_unsafe(&self);
}
```

在这个例子中，`MyUnsafeTrait` 被标记为 `unsafe`，因为它包含一个 `unsafe` 方法 `do_something_unsafe`。

这个方法的具体实现需要在 `unsafe` 块中进行，以确保调用者知道该方法可能会执行不安全的操作。

实现 `unsafe trait` 时，同样需要使用 `unsafe impl` 关键字。例如，我们为一个自定义类型 `MyStruct` 实现 `MyUnsafeTrait`：

```Rust
struct MyStruct;
unsafe impl MyUnsafeTrait for MyStruct {
   unsafe fn do_something_unsafe(&self) {
       // 这里可以执行一些不安全的操作，比如解引用裸指针
       let ptr: *const i32 = std::ptr::null();
       // 注意：这只是一个示例，实际使用时应确保指针有效
       // 此代码会引起panicked
       std::mem::forget(std::ptr::read_volatile(ptr));
   }
}
```

在这个实现中，我们在 `do_something_unsafe` 方法内部执行了一些不安全的操作。由于这些操作绕过了 Rust 的常规安全检查，所以需要使用 `unsafe` 块来包裹，以提醒开发者注意潜在的风险。

### 3.2 安全调用模式

调用 `unsafe trait` 中的方法时，也需要在 `unsafe` 块中进行，以确保调用者清楚了解可能的风险。例如：

```Rust
fn main() {
   let my_struct = MyStruct;

   unsafe {
       my_struct.do_something_unsafe();
   }
}
```

在 `main` 函数中，我们创建了一个 `MyStruct` 的实例，并在 `unsafe` 块中调用了 `do_something_unsafe` 方法。

这样做是为了明确表明该方法的调用可能会导致不安全的行为，从而提醒开发者在调用前确保所有的前置条件都已满足。

在并发编程中，`Send` 和 `Sync` 是两个非常重要的 `unsafe trait`。

`Send` 标记 `trait` 表示实现该 `trait` 的类型可以安全地在不同线程之间传递所有权，而 `Sync` 标记 `trait` 表示实现该 `trait` 的类型可以安全地在多个线程之间共享不可变引用。

例如，大多数基本类型（如 `i32`、`f64` 等）都自动实现了 `Send` 和 `Sync`，因为它们在多线程环境中是线程安全的。然而，对于一些包含裸指针或内部可变状态的类型，可能需要手动实现 `Send` 和 `Sync`，并且在实现时需要特别小心，以确保线程安全性。

```Rust
// 定义一个包含裸指针的结构体
struct MyPtrStruct {
   ptr: *mut i32,
}

// 手动实现 Send trait，这里假设该结构体在线程间传递是安全的
unsafe impl Send for MyPtrStruct {}
// 手动实现 Sync trait，这里假设该结构体在线程间共享是安全的
unsafe impl Sync for MyPtrStruct {}
```

在这个例子中，我们为 `MyPtrStruct` 手动实现了 `Send` 和 `Sync` `trait`。由于该结构体包含裸指针，所以在实现时需要特别小心，确保在多线程环境中不会出现数据竞争或其他安全问题。

## 四、原始指针：Rust 的底层操控利器

原始指针在 Rust 的底层编程中扮演着至关重要的角色，它允许开发者直接操作内存地址，实现一些高效但不安全的操作。在这部分内容中，我们将深入探讨原始指针的各个方面，包括指针类型与转换、指针算术操作以及内存对齐与大小等概念。

### 4.1 指针类型与转换

Rust 中有两种原始指针类型：`*const T`（不可变原始指针）和`*mut T`（可变原始指针）。它们与普通的引用（`&T`和`&mut T`）类似，但原始指针不遵循 Rust 的所有权和借用规则，因此使用时需要格外小心。

创建原始指针非常简单，只需使用`as`操作符将引用转换为原始指针即可。例如：

```Rust
fn main() {
   let num = 5;
   let const_ptr: *const i32 = &num as *const i32;

   let mut num_mut = 10;

   let mut_ptr: *mut i32 = &mut num_mut as *mut i32;
}
```

在这个例子中，我们分别创建了一个指向`num`的不可变原始指针`const_ptr`和一个指向`num_mut`的可变原始指针`mut_ptr`。

需要注意的是，创建原始指针的过程是安全的，因为它只是获取了内存地址，并没有对内存进行读写操作。

然而，当我们需要解引用原始指针，访问它所指向的值时，就需要使用`unsafe`块来确保操作的安全性。例如：

```Rust
fn main() {
   let num = 5;
   let const_ptr: *const i32 = &num as *const i32;

    let mut x = 10;
    let ptr_x = &mut x as *mut i32;
    let y = Box::new(20);
    let ptr_y = &*y as *const i32;
   unsafe {
       *ptr_x += *ptr_y;
       let value = *const_ptr;
       println!("The value pointed by const_ptr is: {}", value);
   }
   assert_eq!(x, 30);
}
```

在这个例子中，我们在`unsafe`块中解引用了`const_ptr`，并打印出了它所指向的值。同时对`ptr_x`指向的值进行了运算，由于解引用原始指针是一种不安全的操作，可能会导致空指针解引用等问题，所以必须在`unsafe`块中进行。


尽管 Rust 在很多场景可以隐式解引用 safe 的指针类型，但原始指针的解引用必须是显式的：

1. . 运算符不会隐式解引用原始指针，你必须用 (*raw).field 或者 (*raw).method(...)。

2. 原始指针并没有实现 Deref，因此强制解引用并不适用于它们。

3. == 和 < 之类的运算符以地址比较原始指针：只有两个原始指针指向同一个内存位置它才是相等的。与此类似，哈希一个原始指针会对它指向的地址进行哈希，而不是对它指向的对象的值进行哈希。

4. 格式化 `trait` 例如 `std::fmt::Dispaly` 会自动解引用，但无法处理原始指针。例外的是 `std::fmt::Debug` 和 `std::fmt::Pointer`，它们会以 16 进制地址的形式显示原始指针，不会解引用它们。

### 4.2 指针算术操作

原始指针支持一些基本的算术操作，如指针偏移、指针比较等。这些操作在底层编程中非常有用，但同样需要在`unsafe`块中进行，以确保操作的安全性。

指针偏移是指通过增加或减少指针的值，使其指向内存中的不同位置。在 Rust 中，可以使用`offset`方法来实现指针偏移。例如：

```Rust
fn main() {
   let arr = [1, 2, 3, 4, 5];

   let ptr: *const i32 = &arr[0] as *const i32;

   unsafe {
       let second_ptr = ptr.offset(1);

       let second_value = *second_ptr;

       println!("The second value in the array is: {}", second_value);
   }
}
```

在这个例子中，我们首先获取了数组`arr`的第一个元素的指针`ptr`，然后使用`offset(1)`方法将指针偏移一个位置，指向数组的第二个元素。最后，我们在`unsafe`块中解引用`second_ptr`，获取并打印出了第二个元素的值。

需要注意的是，指针偏移的步长是根据指针所指向的数据类型的大小来计算的。例如，在上面的例子中，`i32`类型的大小为 4 字节，所以`offset(1)`会将指针偏移 4 个字节。

### 4.3 内存对齐与大小

在计算机中，内存对齐是一种优化技术，它确保数据在内存中的存储位置满足特定的对齐要求，从而提高内存访问的效率。不同的数据类型在内存中具有不同的对齐要求，例如，`i32`类型通常需要 4 字节对齐，而`i64`类型通常需要 8 字节对齐。

Rust 中的原始指针也遵循内存对齐的规则。当我们创建一个原始指针时，它所指向的内存地址必须满足其所指向的数据类型的对齐要求。例如：

```Rust
fn main() {
   let num: i32 = 5;
   let ptr: *const i32 = &num as *const i32;

   // 检查指针的对齐方式
   let alignment = std::mem::align_of_val(&num);

   println!("The alignment of i32 is: {}", alignment);

}
```

在这个例子中，我们使用`std::mem::align_of_val`函数获取了`num`的对齐方式，并打印出了结果。通常情况下，`i32`类型的对齐方式为 4 字节。

了解内存对齐和指针的大小对于编写高效的底层代码非常重要。在进行指针操作时，必须确保指针的对齐方式和所指向的数据类型的对齐要求一致，否则可能会导致未定义行为。

### 4.4 可空指针
Rust 中的空原始指针和 C/C++ 中一样，都是 0 地址。对任何类型 T，`std::ptr::null<T>` 函数会返回一个 `*const T` 空指针，`std::ptr::null_mut<T>` 返回一个 `*mut T` 空指针。

有一些方法可以检查一个原始指针是不是空的。

最简单的是 `is_null` 方法，但 `as_ref` 方法也很方便：

它接受一个 `*const T` 指针然后返回一个 `Option<&'a T>`，把空指针转换为 `None`。

类似的，`as_mut` 方法把一个 `*mut T` 转换成 `Option<&'a mut T>`。

## 五、内存复用黑科技：Union 的高级应用

### 5.1 基础用法示例

`union`是 Rust 中的联合体类型，它允许在同一个内存区域存储不同类型的数据，但在同一时刻只能使用其中一个值。

这与结构体（`struct`）有所不同，结构体的每个字段都有自己独立的内存空间，而`union`的所有字段共享同一块内存，因此`union`的大小由其最大字段的大小决定。

这种特性使得`union`在某些特定场景下非常有用，比如需要对内存进行精细控制或者与 C 语言进行交互时。

下面是一个简单的`union`示例，展示了如何在同一个内存位置存储`i32`和`f32`类型的数据：

```Rust
#[repr(C)]
union MyUnion {
   int_value: i32,
   float_value: f32,
}

fn main() {
   let mut my_union = MyUnion { int_value: 42 };

   unsafe {
       println!("int_value: {}", my_union.int_value);

       my_union.float_value = 3.14;

       println!("float_value: {}", my_union.float_value);
   }
}
```

在这个例子中，我们定义了一个名为`MyUnion`的`union`，它包含两个字段：`int_value`和`float_value`。

`#[repr(C)]`属性用于指定`union`的内存布局与 C 语言中的`union`布局一致，这在与 C 语言代码交互时非常重要。

在`main`函数中，我们创建了一个`MyUnion`的实例`my_union`，并初始化为`int_value`为 42。

然后，我们在`unsafe`块中访问`int_value`字段并打印其值。

接着，我们将`float_value`字段赋值为 3.14，并再次打印其值。需要注意的是，由于`union`的字段共享内存，在同一时间只能安全地访问一个字段，否则可能会导致未定义行为。

### 5.2 安全封装模式

由于 Rust 的类型系统无法静态跟踪`union`当前存储的数据类型，因此直接访问`union`的字段通常需要使用`unsafe`代码块，这增加了出错的风险。

为了提高安全性和易用性，我们可以将`union`封装在一个结构体中，并通过安全的方法来访问和修改`union`的数据。

下面是一个改进后的示例，展示了如何将`union`封装在结构体中，并提供安全的访问方法：

```Rust
#[repr(C)]
union DataUnion {
    int_value: i32,
    float_value: f32,
}

enum DataType {
    Int,
    Float,
}

struct SafeData {
    data: DataUnion,
    data_type: DataType,
}

impl SafeData {
    fn new_int(value: i32) -> Self {
        SafeData {
            data: DataUnion { int_value: value },
            data_type: DataType::Int,
        }
    }

    fn new_float(value: f32) -> Self {
        SafeData {
            data: DataUnion { float_value: value },
            data_type: DataType::Float,
        }
    }

    fn get_int(&self) -> Option<i32> {
        if let DataType::Int = self.data_type {
            unsafe { Some(self.data.int_value) }
        } else {
            None
        }
    }

    fn get_float(&self) -> Option<f32> {
        if let DataType::Float = self.data_type {
            unsafe { Some(self.data.float_value) }
        } else {
            None
        }
    }
}

fn main() {
    let data1 = SafeData::new_int(10);
    let data2 = SafeData::new_float(3.14);

    println!("data1 as int: {:?}", data1.get_int());
    println!("data1 as float: {:?}", data1.get_float());
    println!("data2 as int: {:?}", data2.get_int());
    println!("data2 as float: {:?}", data2.get_float());
}

```

在这个示例中，首先定义了一个`DataUnion`的`union`，用于存储`i32`和`f32`类型的数据。

然后定义了一个`DataType`枚举，用于表示当前`union`中存储的数据类型。

接着，将`DataUnion`和`DataType`封装在`SafeData`结构体中，并为`SafeData`实现了`new_int`和`new_float`方法，用于创建包含不同类型数据的`SafeData`实例。

此外，还实现了`get_int`和`get_float`方法，用于安全地获取`union`中的数据。

在`main`函数中，创建了两个`SafeData`实例，并分别调用`get_int`和`get_float`方法来获取数据，通过这种方式，我们可以确保在访问`union`数据时，类型的一致性和安全性。

## 六、最佳实践与安全准则

在使用`unsafe` Rust 进行编程时，虽然它为我们提供了强大的底层控制能力，但也伴随着更高的风险。

为了确保代码的安全性和可靠性，遵循一些最佳实践和安全准则是至关重要的。

### 6.1 最小化 unsafe 范围

在编写`unsafe`代码时，应尽量将其限制在最小的范围内。只在必要的地方使用`unsafe`关键字，避免不必要的风险。例如，如果你需要调用一个`unsafe`函数，尽量将这个调用封装在一个`unsafe`块中，而不是将整个函数都标记为`unsafe`。

```Rust
fn safe_function() {
   let value = 5;
   let ptr = &value as *const i32;
   // 只在需要解引用指针时使用 unsafe 块
   let result = unsafe { *ptr };
   println!("The result is: {}", result);
}
```

### 6.2 文档化安全契约

对于所有的`unsafe`函数和`unsafe` trait，都应该提供清晰的文档，说明其安全契约。这包括函数的前置条件、后置条件，以及调用者需要满足的要求。例如：

```Rust
/// 从给定的指针读取一个 i32 值。
///
/// # 安全注意事项：
/// - `ptr` 必须是一个有效的、指向 i32 类型的指针。
/// - 调用者必须确保在调用此函数时，`ptr` 不会指向一个已释放的内存区域。
unsafe fn read_i32(ptr: *const i32) -> i32 {
   *ptr
}
```
### 6.3 测试边界条件

对`unsafe`代码进行全面的测试是确保其正确性的关键。

特别是要注意测试各种边界条件，例如空指针、越界指针等。

可以使用 Rust 的测试框架，如`cargo test`，结合`should_panic`属性来测试`unsafe`代码在错误情况下的行为。

```Rust
#[cfg(test)]
mod tests {
   use super::*;

   #[test]
   #[should_panic]
   fn test_read_i32_with_null_pointer() {
       let null_ptr: *const i32 = std::ptr::null();
       unsafe {
           read_i32(null_ptr);
       }
   }
}
```

### 6.4 使用安全抽象

为了减少`unsafe`代码的暴露，可以将其封装在安全的抽象层后面。通过结构体和方法来提供安全的接口，隐藏内部的`unsafe`实现细节。

例如，前面提到的将`union`封装在结构体中，并提供安全的访问方法，就是一种常见的安全抽象模式。

```Rust
struct SafeData {
   data: DataUnion,
   data_type: DataType,
}

impl SafeData {
   fn new_int(value: i32) -> Self {
       SafeData {
           data: DataUnion { int_value: value },
           data_type: DataType::Int,
       }
   }

   fn new_float(value: f32) -> Self {
       SafeData {
           data: DataUnion { float_value: value },
           data_type: DataType::Float,
       }
   }

   fn get_int(&self) -> Option<i32> {
        if let DataType::Int = self.data_type {
            unsafe { Some(self.data.int_value) }
        } else {
            None
        }
    }

    fn get_float(&self) -> Option<f32> {
        if let DataType::Float = self.data_type {
            unsafe { Some(self.data.float_value) }
        } else {
            None
        }
    }
}
```

记住：`unsafe`不是万能药，正确使用才能发挥 Rust 的最大威力。

掌握这些核心能力后，你将能在安全与性能之间找到完美平衡。
