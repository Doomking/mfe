![197605d8-d36f-4347-9fba-e1fa2915aebc_1746803106274269925_origin~tplv-a9rns2rl98-image-qvalue.jpeg](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/4614685ad832403f9bf487eda300497b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1747032598&x-orig-sign=jXyVEBwgJ5g1SdSHDFK%2FwVi0zPo%3D)

- 《Rust Wasm 探索之旅：从入门到实践》系列一：[当 Rust 遇见 WebAssembly：Wasm 与 Rust 生态初探（入门篇）](https://doomking.github.io/mfe/wasm/1-intro.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列二：[Rust+Wasm利器：用wasm-pack引爆前端性能！](https://doomking.github.io/mfe/wasm/2-wasm-pack.html)
- 《Rust Wasm 探索之旅：从入门到实践》系列三：[Rust & WASM 之 `wasm-bindgen` 基础：让 Rust 与 JavaScript 无缝对话](https://doomking.github.io/mfe/wasm/3-wasm-bindgen-base.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列四：[Rust & WASM 之 `wasm-bindgen` 进阶：解锁 Rust 与 JS 的复杂数据交互秘籍](https://doomking.github.io/mfe/wasm/4-wasm-bindgen-advanced.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列五：[Rust & WebAssembly：探索js-sys的奇妙世界](https://doomking.github.io/mfe/wasm/5-wasm-js-sys.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列六：[Rust & WebAssembly：web-sys 开启 DOM 操作新篇](https://doomking.github.io/mfe/wasm/6-wasm-web-sys-base.html)
- 《Rust Wasm 探索之旅：从入门到实践》系列七：[Rust & WebAssembly：`web-sys` 进阶 - 事件处理、异步操作与 Web API 实践](https://doomking.github.io/mfe/wasm/7-wasm-web-sys-advanced.html)
- 《Rust Wasm 探索之旅：从入门到实践》系列八：Rust & WebAssembly 性能调优指南：从毫秒级加速到KB级瘦身

# Rust & WebAssembly 性能调优指南：从毫秒级加速到KB级瘦身

## 一、Wasm 性能影响因素

WebAssembly 作为一种高效的运行时技术，已广泛应用于前端开发中。

然而，要充分发挥Wasm性能优势，还需要关注一些关键因素。在 Rust 与 Wasm 的结合使用中，主要影响性能的因素包括数据拷贝和函数调用开销。

### 数据拷贝：隐形的性能杀手

在 Rust 和 JavaScript 之间传输数据时，通常需要进行数据拷贝。例如，从 Rust 向 JavaScript 传递一个数组时，数据需要从 WebAssembly 的内存中复制到 JavaScript 的内存中。这个过程会增加额外的开销，尤其是在处理大量数据时。根据测试，数据拷贝的开销可能占到总执行时间的 30% 左右。

### 函数调用：跨语言的通信成本

从 JavaScript 调用 Rust 函数或从 Rust 调用 JavaScript 函数时，需要进行上下文切换和参数传递。这种跨语言调用的开销相对较高，尤其是在频繁调用的情况下。

例如，一个简单的 Rust 函数被 JavaScript 每秒调用 1000 次时，调用开销可能占到总时间的 20%。

因此，为了优化性能，我们需要从减少数据传输和优化 Rust 代码本身入手。


## 二、性能优化技巧

### 减少跨语言数据传输

尽量减少 Rust 和 JavaScript 之间的数据交换频率。

例如，可以将多次调用的数据合并为一次调用，减少数据拷贝的次数。

如果需要频繁更新数据，可以考虑将数据存储在 WebAssembly 的内存中，仅在必要时才进行读取或更新, 因此可以通过共享内存来减少这种开销。

使用`WebAssembly.Memory`创建共享内存池，Rust 与 JavaScript 通过指针直接操作同一块内存。

示例如下：

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn init_memory(mem: &mut [u8]) {
   // 假设这里对内存进行初始化操作
   for i in 0..mem.len() {
       mem[i] = i as u8;
   }
}

#[wasm_bindgen]
pub fn process_memory(mem: &mut [u8]) {
   // 模拟对内存数据的处理
   for i in 0..mem.len() {
       mem[i] *= 2;
   }
}
```
在 JavaScript 中，我们可以通过`WebAssembly.Memory`创建共享内存池，然后将内存指针传递给 Rust 函数:

```javascript
import init, { init_memory, process_memory } from './pkg/your_pkg_name.js';

const run = async () => {
    await init();
    const memory = new WebAssembly.Memory({ initial: 1 });
    const buffer = new Uint8Array(memory.buffer);

    init_memory(buffer);
    process_memory(buffer);

    console.log(buffer);
    // 输出: [0, 2, 4, 6, 8, ...]
}

run();
```

### Rust 原生代码优化

优化 Rust 代码本身也是提升 Wasm 性能的关键。

以下是一些优化技巧：

#### 使用合适的数据结构和算法

选择高效的数据结构和算法可以显著提升性能。例如，使用 `Vec` 替代 `LinkedList`，因为 `Vec` 在内存中是连续的，访问速度更快。对于排序操作，优先使用高效的算法，如快速排序或归并排序。

#### 减少不必要的内存分配

避免在循环中进行不必要的内存分配。例如，可以使用 `Vec::with_capacity` 预分配足够的内存，减少动态分配的开销。

示例如下：

```rust
let mut vec = Vec::with_capacity(1000);
for i in 0..1000 {
    vec.push(i);
}
```

#### 利用编译器优化

在编译时启用优化选项可以显著提升代码性能。

在 `Cargo.toml` 中，可以配置 `[profile.release]` 来启用优化：

```toml
[profile.release]
opt-level = 3
lto = true
```

#### 使用内联函数

对于频繁调用的小函数，可以使用 `#[inline]` 属性来减少函数调用的开销：

```rust
#[inline]
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

#### 减少分支预测失败

尽量减少复杂的条件分支，因为分支预测失败会增加执行时间。可以使用模式匹配或循环来替代复杂的条件语句。例如：

```rust
match x {
    1 => println!("One"),
    2 => println!("Two"),
    _ => println!("Other"),
}
```

通过以上优化技巧，可以显著提升 Rust Wasm 的性能，减少数据传输和函数调用的开销。

## 三、利用wasm-opt 优化

`wasm-opt` 是 Binaryen 项目的一部分，它是一个强大的工具，专门用于优化 WebAssembly 文件。通过 `wasm-opt`，可以进一步压缩 Wasm 文件的体积并提升其运行性能。

**安装 `wasm-opt`**：

可以通过 Binaryen 的预构建版本或从源代码编译来安装 `wasm-opt`。

例如，使用 npm 安装 Binaryen 后，就可以直接使用 `wasm-opt`：

```bash
npm install -g binaryen
```

**优化 Wasm 文件**：

使用 `wasm-opt` 对生成的 `.wasm` 文件进行优化。可以通过指定不同的优化级别来调整优化程度。例如，使用 `-O3` 选项可以进行深度优化：

```bash
wasm-opt input.wasm -o output.wasm -O3
```

**优化效果**：

经过 wasm-opt 优化后，Wasm 文件的体积可以显著减小，同时运行速度也会有所提升。根据测试，优化后的 Wasm 文件体积平均可以减少 30% 左右，执行速度可以提升 20% 左右。

例如，一个未优化的 Wasm 文件大小为 1MB，经过 `wasm-opt` 优化后，文件大小可以减少到 700KB 左右。

**优化级别对比：**

| 选项 | 优化方向 | 体积减少 | 速度提升 |
|------|----------|----------|----------|
| `-O1` | 基础优化 | 10-15% | 5-10% |
| `-O3` | 激进优化 | 20-30% | 15-25% |
| `-Os` | 大小优先 | 30-40% | 可变 |
| `-Oz` | 极致压缩 | 40-50% | 可能下降 |

**示例代码**：

假设我们有一个简单的 Rust 代码生成的 Wasm 文件，可以通过以下命令进行优化：

```bash
wasm-opt example.wasm -o example_optimized.wasm -O3
```

优化后的文件可以直接用于 Web 应用，无需额外处理。

通过使用 `wasm-opt`，可以有效地优化 Wasm 文件，减少文件体积并提升性能，这对于提高 Web 应用的加载速度和运行效率至关重要

## 四、代码体积精简术

除了性能优化，控制 WebAssembly 代码的体积同样至关重要。较小的代码体积不仅能加快下载速度，还能减少内存占用，提升应用的整体响应能力。

### wee_alloc：微型分配器的力量

在 Rust 中，默认的全局分配器在 WebAssembly 环境下可能会引入不必要的代码体积。`wee_alloc`作为一个专为 WebAssembly 设计的小巧全局分配器，能够显著减少这部分开销。


在`Cargo.toml`中添加依赖：

```toml
[dependencies]
wee_alloc = "0.4.5"
```

在`lib.rs`中设置为全局分配器：

```rust
use wee_alloc::WeeAlloc;

#[global_allocator]
static ALLOC: WeeAlloc = WeeAlloc::INIT;
```

这样配置后，`wee_alloc`会替代默认分配器，使生成的 Wasm 体积减少约 20KB ，特别适合那些内存操作较少的 WebAssembly 应用场景。

### twiggy：找出体积元凶

`twiggy`是一款强大的工具，用于分析 WebAssembly 文件的大小构成，帮助我们精准定位体积膨胀的源头。

安装`twiggy`：

```bash
cargo install twiggy
```

使用`twiggy top`命令分析`.wasm`文件：

```bash
twiggy top -n 20 your_package_name.wasm
```

它会生成详细报告，展示各函数占用体积的比例，帮助我们识别并移除那些冗余的依赖和代码。其中 `-n 20` 表示只显示体积最大的前 20 个函数，帮助快速定位优化目标。

例如，如果发现某个第三方库函数占用了大量空间，但在实际应用中并未使用，就可以考虑移除该函数或整个库。

**输出示例：**

```
 Shallow Bytes │ Shallow % │ Item
───────────────┼───────────┼─────────────────
         10240 │    45.23% │ "function names" subsection
          5120 │    22.61% │ data[0]
          2048 │     9.04% | custom section "name"
```

### Cargo 配置的体积优化

`Cargo.toml`中的`[profile.release]`部分为我们提供了丰富的优化选项，可在发布模式下有效减小代码体积。

在`Cargo.toml`中设置：

```toml
[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
panic = "abort"
debug = false
```

`opt-level = "z"`侧重于体积优化；`lto = true`开启链接时优化，进一步消除冗余；`codegen-units = 1`减少代码生成单元，提升优化效果；`panic = "abort"`在程序崩溃时直接终止，避免展开调用栈带来的额外代码；`debug = false`关闭调试信息生成，减小文件体积。

### web-sys 的按需引入

`web-sys`库提供了 Rust 与 Web API 的绑定，但默认会引入大量不必要的功能。通过精确指定所需的 API，可以有效减少代码体积。

例如，仅引入`fetch`相关功能：

```toml
[dependencies.web-sys]
version = "0.3"
features = ["Fetch"]
```
这种按需引入的方式能大幅减少绑定代码的体积，有时甚至可减少 50% 以上。

## 总结

在探索 Rust Wasm 的性能优化与代码体积控制的过程中，我们从多个角度深入剖析了影响性能的关键因素，并提供了丰富的优化技巧和工具使用方法。

通过减少 Rust 与 JavaScript 之间的数据传输、优化 Rust 代码本身、利用 `wasm-opt` 工具优化 Wasm 文件，以及采用 `wee_alloc`、`twiggy` 等工具控制代码体积，我们能够显著提升 Rust Wasm 应用的性能和加载速度。
