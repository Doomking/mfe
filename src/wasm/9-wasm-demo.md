![197605d8-d36f-4347-9fba-e1fa2915aebc_1746803106274269925_origin~tplv-a9rns2rl98-image-qvalue.jpeg](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/4614685ad832403f9bf487eda300497b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1747032598&x-orig-sign=jXyVEBwgJ5g1SdSHDFK%2FwVi0zPo%3D)

- 《Rust Wasm 探索之旅：从入门到实践》系列一：[当 Rust 遇见 WebAssembly：Wasm 与 Rust 生态初探（入门篇）](https://doomking.github.io/mfe/wasm/1-intro.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列二：[Rust+Wasm利器：用wasm-pack引爆前端性能！](https://doomking.github.io/mfe/wasm/2-wasm-pack.html)
- 《Rust Wasm 探索之旅：从入门到实践》系列三：[Rust & WASM 之 `wasm-bindgen` 基础：让 Rust 与 JavaScript 无缝对话](https://doomking.github.io/mfe/wasm/3-wasm-bindgen-base.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列四：[Rust & WASM 之 `wasm-bindgen` 进阶：解锁 Rust 与 JS 的复杂数据交互秘籍](https://doomking.github.io/mfe/wasm/4-wasm-bindgen-advanced.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列五：[Rust & WebAssembly：探索js-sys的奇妙世界](https://doomking.github.io/mfe/wasm/5-wasm-js-sys.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列六：[Rust & WebAssembly：web-sys 开启 DOM 操作新篇](https://doomking.github.io/mfe/wasm/6-wasm-web-sys-base.html)
- 《Rust Wasm 探索之旅：从入门到实践》系列七：[Rust & WebAssembly：`web-sys` 进阶 - 事件处理、异步操作与 Web API 实践](https://doomking.github.io/mfe/wasm/7-wasm-web-sys-advanced.html)
- 《Rust Wasm 探索之旅：从入门到实践》系列八：[Rust & WebAssembly 性能调优指南：从毫秒级加速到KB级瘦身](https://doomking.github.io/mfe/wasm/8-wasm-opt.html)

# Rust & WebAssembly 实践：构建一个简单实时的 Markdown 编辑器

## 一、引言：从零打造一个 WebAssembly 应用——实时 Markdown 编辑器
欢迎回到【Rust & WebAssembly】系列！

在前面的八篇文章里，我们已经完成了 Rust 与 WebAssembly 的基础知识和技术、工具的介绍。理解了 Rust 与 WebAssembly 的核心概念，还熟练掌握了编写 WebAssembly 应用的关键工具：用于工程打包的 `wasm-pack`、用于接口绑定的 `wasm-bindgen`、用于 DOM 交互的 `web-sys` 和 `js-sys`。同时介绍了如何进行 性能优化，确保我们的应用产物高效而强大。

那么接下来，我们将通过运用前面所介绍的知识和技术，编写一个可运行的 Rust Wasm 应用 —— 轻量化 Markdown 编辑器。


## 二、项目初始化：从 0 到 1 搭建 Rust Wasm 工程

## （一）环境准备与项目创建

在开启项目之前，先确保已安装好 Rust 工具链。若尚未安装，可前往[Rust](https://www.rust-lang.org/tools/install)[ 官方网站](https://www.rust-lang.org/tools/install)，按照指引完成安装，同时别忘了安装`wasm-pack`，它是我们将 Rust 代码打包成 Wasm 模块的得力助手，使用如下命令即可完成安装：

```bash
cargo install wasm-pack
```
安装成功后，使用`wasm-pack`创建新项目。`wasm-pack`提供了便捷的模板，让我们能快速搭建起标准的 Rust Wasm 项目框架。运行以下命令：
```bash
wasm-pack new markdown-editor
cd markdown-editor
```
### （二）项目结构解析

进入`markdown-editor`项目目录，来看看这个 “框架” 里都有什么。
```
markdown-editor/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   └── utils.rs
├── pkg/
└── tests/
```
- `Cargo.toml`文件，负责管理项目的依赖、版本等重要信息。默认配置了`wasm-bindgen`依赖 ，这是 Rust 与 JavaScript 交互的关键桥梁，后续我们还将在此基础上添加 Markdown 解析库。

- `src`目录是存放项目源代码的地方，其中`lib.rs`是核心文件，Rust 逻辑代码大部分都将放在这里。

- `tests`目录用于存放测试代码.
- `pkg`目录用于存放打包后的 Wasm 模块和相关文件,包括`.wasm`和`.js`等文件。

## 三、引入依赖：配置 Rust 生态工具链

### （一）添加核心依赖
编辑 `Cargo.toml` 文件：
```toml
[package]
name = "markdown-editor"
version = "0.1.0"
authors = ["xxx <xxx@xxx.com>"]
edition = "2018"

[lib]
crate-type = ["cdylib", "rlib"]

[features]
default = ["console_error_panic_hook"]

[dependencies]
wasm-bindgen = "0.2.84"
pulldown-cmark = { version = "0.9", default-features = false }
# The `console_error_panic_hook` crate provides better debugging of panics by
# logging them with `console.error`. This is great for development, but requires
# all the `std::fmt` and `std::panicking` infrastructure, so isn't great for
# code size when deploying.
console_error_panic_hook = { version = "0.1.7", optional = true }

[dependencies.js-sys]
version = "0.3"

[dependencies.web-sys]
version = "0.3"
features = ["Document", "Element", "HtmlElement", "Window", "console"]
[dev-dependencies]
wasm-bindgen-test = "0.3.34"

[profile.release]
# Tell `rustc` to optimize for small code size.
opt-level = "s"

```

依赖如下：
- `wasm-bindgen`: 用于 Rust 和 JavaScript 的交互
- `pulldown-cmark`: 纯 Rust 实现的 Markdown 解析器
- `js-sys` 和 `web-sys`: 提供 JavaScript 和 Web API 的绑定

### （二）配置编译目标

为确保项目能正确编译为 Wasm 模块，需要在`Cargo.toml`中指定输出类型。在文件中添加如下配置：

```toml
[lib]
crate-type = ["cdylib"]
```
这行配置告诉 Rust 编译器，我们要生成一个动态链接库（`cdylib`），这是 Wasm 模块的常见输出类型。有了这个配置，编译器在编译时就会按照 Wasm 的要求生成相应的文件结构和代码，为后续在前端页面中使用 Wasm 模块做好准备 。


## 四、编写核心 Rust 逻辑：打造 Markdown 解析引擎
在`src/lib.rs`文件中，编写核心的 Markdown 解析函数。首先，引入必要的库和模块, 然后再编写 `render_markdown` 函数来实现 Markdown 解析：
```rust
use js_sys::{Error, JsString};
use pulldown_cmark::{html, Options, Parser};
use wasm_bindgen::prelude::*;
use web_sys::console;

// 当编译 Wasm 时启用 console_error_panic_hook
#[cfg(feature = "console_error_panic_hook")]
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn error(s: &str);
}

/// 将 Markdown 文本转换为 HTML
#[wasm_bindgen]
pub fn render_markdown(input: &str) -> Result<JsString, JsValue> {
    // 设置 panic hook 以便更好的错误提示
    #[cfg(feature = "console_error_panic_hook")]
    console_error_panic_hook::set_once();

    console::log_1(&"开始渲染 Markdown...".into());

    // 配置 Markdown 解析选项
    let mut options = Options::empty();
    options.insert(Options::ENABLE_STRIKETHROUGH);
    options.insert(Options::ENABLE_TABLES);
    options.insert(Options::ENABLE_FOOTNOTES);
    options.insert(Options::ENABLE_TASKLISTS);

    // 创建解析器
    let parser = Parser::new_ext(input, options);

    // 渲染为 HTML
    let mut html_output = String::new();
    html::push_html(&mut html_output, parser);

    console::log_1(&"Markdown 渲染完成".into());

    Ok(JsString::from(html_output))
}

/// 初始化 Wasm 模块
#[wasm_bindgen(start)]
pub fn init() -> Result<(), JsValue> {
    // 设置 panic hook
    #[cfg(debug_assertions)]
    console_error_panic_hook::set_once();

    console::log_1(&"Markdown 编辑器 Wasm 模块已初始化".into());

    Ok(())
}

```
- `Parser::new_ext(input, options)` 创建了一个 `Parser` 实例，它就像是一个 “文本处理器”，能够对输入的 Markdown 文本进行流式处理，将其转换为一个可迭代的事件流。
- `html::push_html(&mut html_output, parser)` 则将解析后的 Markdown 内容以 HTML 格式推送到`html_output`字符串中，最终返回生成的 HTML 字符串，完成 Markdown 到 HTML 的华丽转变 。

## 五、构建 Wasm 模块：从 Rust 代码到二进制资产

### （一）执行构建命令

完成 Rust 核心逻辑的编写后，接下来就需要使用 `wasm-pack` 将 Rust 代码编译成浏览器能够识别和运行的 WebAssembly 模块。进入项目根目录，运行以下命令：
```bash
wasm-pack build --target web
```

这里的`--target web`参数明确指定了输出格式为适用于浏览器环境的模块。在编译过程中，`wasm-pack`会根据这个参数生成符合 ES6 模块规范的 JavaScript 胶水代码。

当命令执行完毕后，项目目录中多了一个`pkg/`目录，这里就是存放构建产物的地方。

在`pkg/`目录下，有两个关键文件：`markdown_editor.js` 和 `markdown_editor_bg.wasm`。它们是一对紧密合作的 “伙伴”，共同为前端页面提供强大的 Markdown 解析功能 。

### （二）构建产物解析

- `markdown_editor_bg.wasm` 文件是二进制格式的 WebAssembly 模块，它包含了编译后的 Rust 代码，是整个项目的核心计算逻辑所在。它以紧凑的二进制形式存储，在浏览器中加载和执行时，能够以接近原生的速度运行，为 Markdown 解析提供了强大的性能支持 。

- `markdown_editor.js` 则是自动生成的 JavaScript 接口层，它充当了 Rust 与 JavaScript 之间的 “翻译官”。这个文件封装了 Wasm 模块的加载和函数调用逻辑，使得 JavaScript 代码能够方便地加载 `rust_markdown_editor_bg.wasm` 模块，并调用其中由 `#[wasm_bindgen]` 导出的 `render_markdown` 函数。`rust_markdown_editor.js`会处理好数据类型的转换、函数参数的传递等复杂细节，让前端开发者可以像调用普通 JavaScript 函数一样调用 Rust 编写的 Markdown 解析逻辑 ，极大地降低了开发的难度和复杂性 。

- `markdown_editor.d.ts` 是 TypeScript 类型定义文件，它为前端 JavaScript 代码提供了类型信息，使得在开发过程中能够获得更好的代码提示和类型检查 。


## 六、创建前端界面：搭建交互层与视图层

有了强大的 Markdown 解析能力，再加上前端界面，用户就能够与我们的应用进行交互了。

接下来我们将创建一个简单直观的 HTML 页面，并编写 JavaScript 代码来加载 Wasm 模块，实现 Markdown 文本的实时预览功能。

在项目根目录下（与 `src` 和 `pkg` 同级），创建一个 `index.html` 文件：
```html
<!doctype html>
<html lang="zh-CN">
    <head>
        <meta charset="UTF-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <title>Rust Wasm Markdown 编辑器</title>
        <style>
            body {
                font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
                max-width: 1200px;
                margin: 0 auto;
                padding: 20px;
                background-color: #f5f5f5;
            }
            .container {
                display: flex;
                gap: 20px;
                height: 80vh;
            }
            .editor-panel,
            .preview-panel {
                flex: 1;
                background: white;
                border-radius: 8px;
                box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
                padding: 20px;
                overflow: auto;
            }
            textarea {
                width: 100%;
                height: 100%;
                border: none;
                resize: none;
                font-family: "Monaco", "Menlo", "Ubuntu Mono", monospace;
                font-size: 14px;
                outline: none;
            }
            h1 {
                color: #333;
                text-align: center;
                margin-bottom: 30px;
            }
            .panel-title {
                font-weight: bold;
                margin-bottom: 10px;
                color: #555;
            }
        </style>
    </head>
    <body>
        <h1>Rust Wasm Markdown 编辑器</h1>

        <div class="container">
            <div class="editor-panel">
                <div class="panel-title">编辑区 (Markdown)</div>
                <textarea id="editor" placeholder="请输入 Markdown 文本...">
# Hello Rust Wasm!
            </textarea
                >
            </div>

            <div class="preview-panel">
                <div class="panel-title">预览区 (HTML)</div>
                <div id="preview"></div>
            </div>
        </div>

        <script type="module">
            import init, { render_markdown } from "./pkg/markdown_editor.js";

            async function run() {
                // 初始化 Wasm 模块
                await init();

                const editor = document.getElementById("editor");
                const preview = document.getElementById("preview");

                // 初始渲染
                updatePreview();

                // 监听输入事件
                editor.addEventListener("input", updatePreview);

                function updatePreview() {
                    try {
                        const markdownText = editor.value;
                        const html = render_markdown(markdownText);
                        preview.innerHTML = html;
                    } catch (error) {
                        preview.innerHTML = `<div style="color: red;">渲染错误: ${error}</div>`;
                    }
                }
            }

            run().catch(console.error);
        </script>
    </body>
</html>

```
`updatePreview` 实现了实时预览功能，当用户在编辑区输入 Markdown 文本时，会触发`input`事件，调用`updatePreview`函数更新预览区的内容。

这种设计模式使得用户在输入 Markdown 内容的同时，能够即时看到转换后的 HTML 效果，大大提升了用户体验。

## 七、运行与部署：从本地调试到生产环境

### （一）本地运行
完成以上步骤，就可以运行我们的 Markdown 编辑器应用了。

运行应用的第一步是启动一个静态文件服务器，将我们编写的 HTML、JavaScript 和 Wasm 文件提供给浏览器访问。

```bash
# 使用 Python 简单服务器
python3 -m http.server 8080

# 或者使用 Node.js 的 http-server
npx http-server
```

然后在浏览器中打开 `http://localhost:8080` 就能看到你的 Markdown 编辑器了！

![效果图](./demo_09/WX20250830-234826@2x.png)

### （二）部署优化（可选）

如果想要将应用部署到生产环境，还需要进行一些优化，以提升应用的性能和稳定性。

在构建 Wasm 模块时，添加`--release`标志，以启用优化并减少调试信息。

```
wasm-pack build --release --target web
```

添加 `--release` 标志后，`wasm-pack` 会在构建过程中对代码进行一系列优化，如去除不必要的调试信息、优化函数调用、减少代码体积等。

这些优化措施可以显著提升 Wasm 模块的执行效率，让应用在生产环境中运行得更加流畅。

例如，在处理大段 Markdown 文本时，经过优化的 Wasm 模块能够更快地完成解析和渲染，减少用户等待的时间。同时，去除调试信息也可以减小 Wasm 文件的大小，加快文件的加载速度，进一步提升用户体验。

## 八、总结：Rust Wasm 实战中的技术价值

### （一）工具链价值

| 工具/库        | 作用一句话                     |
|----------------|--------------------------------|
| `wasm-pack`    | 一条龙生成可发布包             |
| `wasm-bindgen` | Rust ↔ JS 的翻译官             |
| `web-sys/js-sys` | DOM 操作和 JS 交互            |
| `pulldown-cmark` | 纯 Rust Markdown 解析器       |


### （二）Rust Wasm 的核心优势

1.  **性能优势**：Rust 编译生成的 Wasm 代码，在执行效率上表现卓越，近乎原生。Rust 编译器基于 LLVM，能够生成高度优化的机器码，使得 Wasm 模块在浏览器中能够快速执行，大大缩短了 Markdown 文本的解析时间，为用户带来了即时的预览反馈 。

2.  **内存安全**：Rust 引以为傲的所有权系统，在编译期就对内存访问进行了严格的检查和管理，从根本上杜绝了空指针引用、数据竞争等常见的内存安全问题。这种内存安全特性，不仅提升了应用的可靠性，还减少了开发过程中的调试成本，让开发者可以更加专注于功能的实现和优化。

3.  **代码复用**：借助 Rust Wasm，我们实现了代码的高效复用。在项目中，Markdown 解析的核心逻辑既可以在前端的 Wasm 模块中运行，为用户提供实时预览功能；也可以在后端的 Rust API 服务中复用，用于服务器端的 Markdown 内容处理。


## 结语

这个 Markdown 编辑器虽然简单，但完整展示了 Rust WebAssembly 的开发全流程。因此你可以在此基础上继续扩展功能，比如添加代码高亮、导出功能、历史版本等。