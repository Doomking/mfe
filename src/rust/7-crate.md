![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust 的 crate 和模块：强大的代码组织工具


# 一、Rust 的 crate 和模块

Rust 程序由crates 组成。

crate 是 Rust 在**编译时最小的代码单位**，每一个crate 都是一个完整的、一体的单元，有两种形式：二进制项和库。

模块（Module）则是用于**组织代码的结构**，类似命名空间-包含函数、类型、常量等内容的容器，模块组成了Rust的程序或库。

# 二、crate 的定义与基本原理

## （一）crate 的概念

crate 在 Rust 中是编译时最小的代码单位。它主要有两种形式：二进制项和库。

- 二进制项：可以被编译为可执行程序，比如一个命令行程序或者一个服务器。它必须有一个 main 函数。目前很多项目都是以二进制项的形式存在。例如，可以使用 cargo new my_binary --bin 创建一个名为 my_binary 的二进制 crate 项目。

- 库 crate ：没有 main 函数，不会编译为可执行程序，而是提供一些诸如函数之类的东西。在 Rust 社区中，库 crate 通常被设计用于满足特定需求或解决特定问题，比如提供数学计算、网络通信等功能。

项目中，crate 的作用至关重要。它可以将相关的代码组织在一起，形成一个独立的编译单元。通过合理地划分和组织 crate，可以提高代码的可维护性和可重用性。

## （二）包与 crate 的关系

包（package）是由一个或多个 crate 组成的。一个包会包含一个 Cargo.toml 文件，这个文件描述了如何去构建这些 crate。

包中可以包含至多一个库 crate，库 crate 可以提供可重用的代码，供其他 crate 引用。同时，包中可以包含任意多个二进制 crate。二进制 crate 是可执行的程序，它们可以直接运行。

包的管理方式主要通过 Cargo 工具来实现。Cargo 会根据 Cargo.toml 文件中的配置信息，自动下载和构建包所依赖的 crate。在一个包中，如果同时存在 src/main.rs 和 src/lib.rs，那么这个包就有两个 crate：一个二进制的和一个库的，且名字都与包相同。

通过将文件放在 src/bin 目录下，一个包可以拥有多个二进制 crate，每个 src/bin 下的文件都会被编译成一个独立的二进制 crate。

# 三、模块的定义与基本原理

## （一）模块的概念与结构

在 Rust 中，模块是一种用于组织代码的机制，可以将相关的函数、结构体、枚举和常量等内容封装在一起。模块内部的结构可以根据需要灵活组织，例如，可以将相关的函数放在同一个模块内，将不同的功能组织在不同的模块中，以便更好地管理和维护代码。

一个模块内部可以包含以下内容：

-   函数：定义和实现特定功能的函数。

-   结构体：定义和实现自定义的数据结构。

-   枚举：定义和实现枚举类型。

-   常量：定义和实现常量值。

模块可以嵌套，形成层次结构。例如，可以在一个模块内部定义另一个模块，从而实现更复杂的代码组织：
```rust
mod my_module {
    // 函数定义
    pub fn greet() {
        println!("Hello, world!");
    }
    // 结构体定义
    pub struct Person {
        name: String,
        age: u32,
    }
    // 枚举定义
    pub enum Color { Red, Green, Blue, }
}
```
## （二）模块的访问控制

在 Rust 中，模块提供了访问控制的机制，可以限制模块内部的内容对外的可见性。通过使用 pub 关键字，可以指定哪些内容对外可见：
```rust
mod my_module {
    // 公开函数
    pub fn greet() {
        println!("Hello, world!");
    }
    // 私有函数
    fn secret_function() {
        println!("This is a secret function.");
    }
}
```
上述示例中， pub 关键字将 greet 函数标记为公开的，可以在模块外部访问。而 secret_function 函数没有使用 pub 关键字，因此它是私有的，只能在模块内部访问。

## （三）模块的使用方法

在 Rust 中，可以使用 use 关键字引入模块和其内部的内容，以便在其他地方直接使用：
```rust
mod my_module {
    pub fn greet() {
        println!("Hello, world!");
    }
}
// 在其他地方使用模块和函数
use my_module::greet;
fn main() {
    greet(); // 调用模块内部的函数
}
```
此外，还可以使用绝对路径或相对路径来引入模块。绝对路径以包名或 crate 作为开头，从包根开始查找；相对路径以当前模块开始，以 self、super 或当前模块的标识符作为开头：
```rust
mod outer_module {
    mod inner_module {
        pub fn inner_function() {
            println!("This is an inner function.");
        }
    }
}
fn main() {
    // 绝对路径引入
    crate::outer_module::inner_module::inner_function();
    // 相对路径引入
    use outer_module::inner_module;
    inner_module::inner_function();
}
```
# 四、crate 和模块的关系

use crate::是 Rust 语言中的一个模块导入语句。在 Rust 中，模块是用来组织代码的一种方式，允许你将相关的函数、类型等组合在一起，并可以从其他模块中导入和使用它们。

一个 crate 通常对应于一个库或应用程序。它可以包含多个模块，模块之间通过路径来组织。

路径在模块之间的组织中起着关键作用。use语句后面跟的路径指定了要导入的内容的位置。

crate::是这个路径的一部分，表示从当前 crate 的根开始。例如，use crate::my_module::my_function;会导入my_module中定义的my_function函数。这

Rust 的这种模块组织方式使得代码更加清晰、易于维护。通过合理地划分模块和使用路径，可以将复杂的应用程序分解为多个较小的、可管理的部分。每个模块可以专注于特定的功能，并且可以独立开发、测试和维护。

同时，通过use语句和路径，可以方便地在不同的模块之间共享代码和功能。

此外，别名的使用也可以在处理模块导入时提供更大的灵活性。例如，use crate::my_module::MyType as AnotherName；会将MyType导入并重命名为AnotherName，这在避免命名冲突或使代码更具可读性时很有用。

# 五、测试与文档

## （一）测试方法

- 断言：Rust 提供了多种断言宏，如 assert_eq!、assert_ne!、assert! 等。这些宏可以帮助开发者在测试中验证代码的行为是否符合预期。例如，assert_eq!(add(1, 2), 3) 用于验证 add 函数的结果是否为 3。

- 单元测试： Rust 中，单元测试通常放在一个带有 #[cfg(test)] 属性的模块中，测试函数要加上 #[test] 属性：

- #[test] fn test_add() { assert_eq!(add(1, 2), 3); }。单元测试可以覆盖多种情况，包括边界值、异常值等，以确保代码的健壮性。

- 集成测试：集成测试通常放在项目根目录下的 tests 目录中。每个测试文件都是一个独立的 crate，可以模拟外部依赖，如数据库、网络请求等，以隔离测试环境，确保测试结果的准确性。

## （二）文档的编写方式

在 Rust 中，文档的编写方式有多种。

- 文档元素级编写方式：对于函数、结构体等代码的公共 API，都应该有文档。文档应该包括一个简短的句子解释其功能，更详细的解释，至少一个可复制粘贴的代码示例，以及必要的更高级的解释。
```rust
use std::env;
// Prints each argument on a separate line
for argument in env::args() {
    println!("{argument}");
}
```
- 模块级文档编写方式：除了函数等的文档，包和模块也可以添加注释。包级别的注释分为行注释 //! 和块注释 /*!... */，放在包、模块的最上方。例如：
```rust
/*! lib包是world_hello二进制包的依赖包，里面包含了compute等有用模块 */
pub mod compute;
```
- 文档属性：Rust 的文档有一些属性可以使用。例如，#[doc(inline)] 用于内联文档，而不是链接到单独的页面；#[doc(no_inline)] 用于防止链接到单独的页面或其他位置；#[doc(hidden)] 用于告诉 rustdoc 不要包含此项到文档中。

- 文档化测试：Rust 的文档注释中可以写测试用例，省去了单独写测试用例的环节：
```rust
/// `add_one` 将指定值加1
///
/// # Examples
///
/// ```
/// let arg = 5;
/// let answer = my_crate::add_one(arg);
///
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```
# 六、依赖管理

## （一）依赖的添加方式

在 Rust 中，依赖管理主要通过 Cargo 工具来实现。Cargo 提供了多种方式来添加依赖：

- 从 [crates.io](http://crates.io) 添加依赖：

    [crates.io](http://crates.io)：是 Rust 的包注册表，类似于其他语言的包管理仓库。在 Rust 项目中，我们可以在 Cargo.toml 文件中添加来自 [crates.io](http://crates.io) 的依赖。[](http://crates.io)

    例如，要添加一个名为 ferris-says 的依赖，可以在 Cargo.toml 文件的 [dependencies] 部分添加以下内容：ferris-says = "0.3.1"。也可以通过运行 cargo add ferris-says 命令来添加依赖。Cargo 会自动为我们下载和安装这个依赖。

- 从 git 仓库添加依赖：

    如果我们需要从一个 git 仓库添加依赖，可以在 Cargo.toml 文件中指定仓库的地址和相关信息。

    例如，[dependencies]rand = { git = "[https://github.com/rust-lang-nursery/rand"}，](https://github.com/rust-lang-nursery/rand)在这种情况下，Cargo 会从指定的 git 仓库下载依赖，并根据仓库中的配置进行构建。

- 从本地路径添加依赖：

    如果依赖是一个本地的 Rust 项目，可以通过相对路径来添加依赖。

    例如，[dependencies]bar = { path = "../bar" }。这里假设 bar 项目位于当前项目的上级目录中。Cargo 会将本地路径的项目作为依赖进行构建，并在当前项目中使用。

## （二）版本控制与条件依赖

- 语义化版本控制：

    Rust 的包管理工具 Cargo 支持语义化版本控制（Semantic Versioning）。语义化版本控制将版本号分为主版本号、次版本号和修订号，格式为 MAJOR.MINOR.PATCH。

    例如，在 Cargo.toml 中指定依赖版本时，可以使用特殊符号来表示对依赖版本的需求。比如，[dependencies]my_dependency = "^0.1"，这将匹配 my_dependency 的 0.1.x 系列版本。

- 条件依赖：

    在实际项目中，可能需要根据不同的条件来添加依赖。Cargo 支持在 Cargo.toml 中使用条件依赖。

    例如，只在 Linux 系统上使用某个特定的库，可以这样写：[dependencies]my_dependency = { version = "0.1.0", features = ["linux"] }。这里的 features 字段是一个布尔值数组，表示这个依赖是否需要特定的功能。在这个例子中，我们只希望在 Linux 系统上使用这个库。

- 环境变量在依赖管理中的作用：

    Cargo 还允许使用环境变量来控制依赖的版本。

    例如，设置一个名为 CARGO_FEATURE_MY_DEPENDENCY 的环境变量，然后在 Cargo.toml 中使用它：[dependencies]my_dependency = { version = "0.1.0", features = ["$CARGO_FEATURE"] }。

    这样，只有当环境变量 CARGO_FEATURE_MY_DEPENDENCY 被设置为某个值时，my_dependency 的特定功能才会被启用。

# 七、工作空间

## （一）工作空间的概念

在 Rust 中，工作空间是一种用于管理多个相关包的机制。它允许开发者将多个 crate 组织在一起，方便进行开发、测试和构建。工作空间可以看作是一个包含多个 crate 的目录结构，每个 crate 都可以独立开发和构建。

## （二）工作空间的结构

一个工作空间通常由一个根目录和多个 crate 目录组成。根目录下包含一个 Cargo.toml 文件，用于描述整个工作空间的配置信息。每个 crate 目录下也有一个自己的 Cargo.toml 文件，用于描述该 crate 的配置信息。

一个简单的工作空间结构可能如下：
```rust
my_workspace/
    ├── Cargo.toml
    ├── crate1/│
        ├── Cargo.toml│
        └── src/│
            └── main.rs
    ├── crate2/
        ├── Cargo.toml
            └── src/
                └── lib.rs
```
在这个例子中，my_workspace 是工作空间的根目录，包含两个 crate：crate1 和 crate2。每个 crate 都有自己的 Cargo.toml 文件和源代码目录。

## （三）工作空间的优势

1.  方便管理多个相关的 crate：工作空间可以将多个相关的 crate 组织在一起，方便进行开发和管理。开发者可以在一个工作空间中同时开发多个 crate，而不需要在多个独立的项目之间切换。
1.  共享依赖：工作空间中的多个 crate 可以共享依赖。如果多个 crate 都需要使用同一个依赖，只需要在工作空间的根目录下的 Cargo.toml 文件中添加一次依赖即可。这样可以减少重复的依赖管理工作，提高开发效率。
1.  统一构建和测试：工作空间可以统一构建和测试多个 crate。开发者可以在工作空间的根目录下运行 cargo build 和 cargo test 命令，来构建和测试整个工作空间中的所有 crate。这样可以方便地进行集成测试，确保多个 crate 之间的交互和协作正常。

### （四）工作空间的使用方法

1.  创建工作空间：可以使用 cargo new --workspace my_workspace 命令来创建一个名为 my_workspace 的工作空间。这个命令会在当前目录下创建一个工作空间的根目录，并在根目录下生成一个 Cargo.toml 文件。
1.  添加 crate 到工作空间：可以使用 cargo new crate1 和 cargo new crate2 命令在工作空间的根目录下创建两个 crate：crate1 和 crate2。这些 crate 会自动被添加到工作空间中。
1.  构建和测试工作空间：可以在工作空间的根目录下运行 cargo build 和 cargo test 命令来构建和测试整个工作空间中的所有 crate。
