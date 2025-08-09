![197605d8-d36f-4347-9fba-e1fa2915aebc_1746803106274269925_origin~tplv-a9rns2rl98-image-qvalue.jpeg](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/4614685ad832403f9bf487eda300497b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1747032598&x-orig-sign=jXyVEBwgJ5g1SdSHDFK%2FwVi0zPo%3D)

- 《Rust Wasm 探索之旅：从入门到实践》系列一：[当 Rust 遇见 WebAssembly：Wasm 与 Rust 生态初探（入门篇）](https://doomking.github.io/mfe/wasm/1-intro.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列二：[Rust+Wasm利器：用wasm-pack引爆前端性能！](https://doomking.github.io/mfe/wasm/2-wasm-pack.html)
- 《Rust Wasm 探索之旅：从入门到实践》系列三：[Rust & WASM 之 `wasm-bindgen` 基础：让 Rust 与 JavaScript 无缝对话](https://doomking.github.io/mfe/wasm/3-wasm-bindgen-base.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列四：[Rust & WASM 之 `wasm-bindgen` 进阶：解锁 Rust 与 JS 的复杂数据交互秘籍](https://doomking.github.io/mfe/wasm/4-wasm-bindgen-advanced.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列五：[Rust & WebAssembly：探索js-sys的奇妙世界](https://doomking.github.io/mfe/wasm/5-wasm-js-sys.html)

- 《Rust Wasm 探索之旅：从入门到实践》系列六：[Rust & WebAssembly：web-sys 开启 DOM 操作新篇](https://doomking.github.io/mfe/wasm/6-wasm-web-sys-base.html)
- 《Rust Wasm 探索之旅：从入门到实践》系列七：Rust & WebAssembly：`web-sys` 进阶 - 事件处理、异步操作与 Web API 实践

# `web-sys` 进阶：事件处理、异步操作与 Web API 实践

## 引言

上一篇文章中，我们步了解了如何使用 `web-sys` 在 Rust 中操作浏览器的 DOM。

那么接下来，我们将更进一步，探索 `web-sys` 的高级功能，包括复杂的**事件处理**、**异步操作**、**网络请求**、**绘图操作**、**本地存储**以及**定时器**的使用。

这些功能将帮助开发者更全面地利用 Rust 和 WebAssembly 的强大能力，构建复杂的 Web 应用。

## 一、复杂的事件处理

在 Web 开发中，事件处理是实现交互性的关键。通过处理各种事件，可以让网页响应用户的操作，如点击按钮、输入文本等。

在 Rust 与 WebAssembly 的开发中，借助`web-sys`库，开发者能够方便地处理各种浏览器事件。

### 事件类型与对象

`web-sys` 支持多种事件类型，例如 `click`、`mousemove`、`keydown` 等。每个事件类型都有对应的事件对象，例如 `MouseEvent` 和 `KeyboardEvent`。

```rust
use wasm_bindgen::prelude::*;
use web_sys::{window, EventTarget,MouseEvent, console};
use js_sys::Closure;

// 添加鼠标移动事件
#[wasm_bindgen]
pub fn add_mouse_move() {
    let document = window().unwrap().document().unwrap();
    let container = document.get_element_by_id("my_container").unwrap();

    let callback = Closure::<dyn FnMut(_)>::new(move |event: MouseEvent| {
        let message = format!(
            "Mouse moved to ({}, {})",
            event.client_x(), // 获取鼠标 X 坐标
            event.client_y()  // 获取鼠标 Y 坐标
        );
        console::log_1(&message.into());
    });

    container
        .add_event_listener_with_callback("mousemove", callback.as_ref().unchecked_ref())
        .unwrap();
    callback.forget();
}


// 添加点击事件
#[wasm_bindgen]
pub fn add_click() {
    let document = window().unwrap().document().unwrap();
    let button = document.get_element_by_id("my_button").unwrap();
    let event_target: EventTarget = button.dyn_into().unwrap();

    let closure = Closure::wrap(Box::new(move |event: web_sys::MouseEvent| {
        console::log_1(
            &format!(
                "Button clicked at x: {}, y: {}",
                event.client_x(),
                event.client_y()
            )
            .into(),
        );
    }) as Box<dyn FnMut(_)>);

    event_target
        .add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())
        .unwrap();

    closure.forget();
}
```

上述代码中，分别为DOM元素添加了`mousemove`和`click`事件。

### 添加及移除事件监听

在某些情况下，需要移除已经添加的事件监听，以避免内存泄漏和不必要的事件触发。在 Rust 中，可以使用`remove_event_listener`方法来移除事件监听。

以下是一个添加和移除事件监听的示例代码：

```rust
use std::{cell::RefCell, rc::Rc};

use wasm_bindgen::prelude::*;
use js_sys::Function;
use web_sys::{window, EventTarget, console};

#[wasm_bindgen]
pub fn add_click_with_return_remove() -> Function {
    let document = window().unwrap().document().unwrap();
    let button = document.get_element_by_id("my_button").unwrap();
    let event_target: EventTarget = button.dyn_into().unwrap();

    // 使用 Rc 和 RefCell 实现共享所有权
    let closure = Rc::new(RefCell::new(None));
    let closure_clone = Rc::clone(&closure);
    
    // 创建事件处理闭包
    let handler = Closure::wrap(Box::new(move |_event: web_sys::MouseEvent| {
        console::log_1(&"Button clicked!".into());
    }) as Box<dyn FnMut(_)>);

    // 添加事件监听器
    event_target
        .add_event_listener_with_callback("click", handler.as_ref().unchecked_ref())
        .unwrap();

    // 将闭包存储在 Rc 中
    *closure_clone.borrow_mut() = Some(handler);

    // 创建并返回清理函数
    Closure::<dyn FnMut()>::new(move || {
        if let Some(handler) = closure.borrow_mut().take() {
            // 移除事件监听器
            let _ = event_target.remove_event_listener_with_callback(
                "click", 
                handler.as_ref().unchecked_ref()
            );
            
            // 显式释放闭包内存
            // handler.forget();
            console::log_1(&"Event listener removed!".into());
        }
    }).into_js_value().unchecked_into()
}
```

上述函数调用后，为`my_button`元素添加了`click`事件，并返回一个移除事件的函数，用于移除添加的事件；

在 JavaScript 中调用代码如下：

```javascript
import init, { add_click_with_return_remove } from './pkg/your_pkg_name.js';

const run = async () => {
    await init();
    // 添加my_button的click事件
    const remove = add_click_with_return_remove(); // 假设有个id为'my_button'的button
    console.log("remove function: ", remove); 
    const $removeBtn = document.getElementById('remove_button'); // 假设有个id为'remove_button'的button
    $removeBtn.addEventListener('click', () => {
        // 移除my_button的click事件
        remove();
    });
};

run();
```

### 事件委托模式

通过在 `document` 上添加 `click` 事件，并根据 `EventTarget` 来判断点击了不同的元素，从而触发不同的事件处理逻辑来实现事件委托。

```rust
#[wasm_bindgen]
pub fn add_delegate() {
    let document = window().unwrap().document().unwrap();
    let closure = Closure::wrap(Box::new(move |event: web_sys::MouseEvent| {
        let target = event.target().unwrap();
        if let Some(element) = target.dyn_ref::<HtmlElement>() {
            match element.id().as_str() {
                "btn-save" => {
                    console::log_1(
                        &format!(
                            "btn-save Button clicked at x: {}, y: {}",
                            event.client_x(),
                            event.client_y()
                        )
                        .into(),
                    );
                }
                "btn-delete" => {
                    console::log_1(
                        &format!(
                            "btn-delete Button clicked at x: {}, y: {}",
                            event.client_x(),
                            event.client_y()
                        )
                        .into(),
                    );
                }
                _ => {}
            }
        }
    }) as Box<dyn FnMut(_)>);

    document
        .add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())
        .unwrap();

    closure.forget();
}
```

## 二、监听DOM变化

如果想要监听DOM变化，可以使用 `MutationObserver` 来实现。 `MutationObserver` 是一个浏览器提供的 API，用于监听 DOM 树的变化。它可以在 DOM 元素被添加、删除、修改或属性发生变化时触发回调函数。

以下是一个使用 `MutationObserver` 监听 DOM 变化的示例代码：

```rust
use js_sys::Function;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use js_sys::Array;
use web_sys::{console, window, Element, MutationObserver, MutationObserverInit};

/// 一个辅助函数，用于在异步函数中暂停指定的毫秒数。
async fn sleep(ms: i32) {
    let promise = js_sys::Promise::new(&mut |resolve: Function, _| {
        window()
            .unwrap()
            .set_timeout_with_callback_and_timeout_and_arguments_0(
                &resolve,
                ms,
            )
            .unwrap();
    });
    JsFuture::from(promise).await.unwrap();
}

// 核心功能：创建一个结构体来将 Observer 和 Callback 绑定在一起。
// 这样开发者就可以利用 Rust 的 Drop trait 来自动清理资源。
struct DomObserver {
    observer: MutationObserver,
    // _callback 虽然没有被直接读取，但必须被这个结构体拥有，
    // 以确保它的生命周期与 observer 一致。
    _callback: Closure<dyn FnMut(Array, MutationObserver)>,
}

impl Drop for DomObserver {
    // 当 DomObserver 实例离开作用域时，drop 方法会被自动调用。
    fn drop(&mut self) {
        // 在这里断开观察，可以确保不会有遗漏。
        self.observer.disconnect();
        console::log_1(&"Observer disconnected and memory cleaned up automatically.".into());
    }
}

/// 观察指定ID的DOM元素，并在一段时间后自动停止。
///
/// # Arguments
/// * `element_id` - 要观察的元素的ID。
/// * `duration_ms` - 观察持续的时间（毫秒）。
#[wasm_bindgen]
pub async fn observe_dom_changes(element_id: String, duration_ms: i32) -> Result<(), JsValue> {
    let window = window().expect("should have a window");
    let document = window.document().expect("should have a document");
    let target: Element = document
        .get_element_by_id(&element_id)
        .ok_or_else(|| JsValue::from_str(&format!("Element with id '{}' not found", element_id)))?
        .dyn_into()?;

    // 1. 创建回调函数。
    // MutationObserver 的回调接收两个参数：一个 MutationRecord 数组和一个观察者实例。
    let callback = Closure::new(move |mutations: Array, _observer: MutationObserver| {
        console::log_1(&"DOM changes detected!".into());
        // 开发者可以遍历变化的具体内容
        for mutation in mutations.iter() {
            console::log_2(&"  - Mutation:".into(), &mutation);
        }
    });

    // 2. 使用 web-sys 内置的构造函数创建 MutationObserver。
    let observer = MutationObserver::new(callback.as_ref().unchecked_ref())?;

    // 3. 将 observer 和 callback 存入开发者自定义的结构体中。
    //    现在它们的生命周期被绑定在了一起。
    let observer_handle = DomObserver {
        observer,
        _callback: callback,
    };
    
    // 4. 配置要观察的变化类型。
    let config = MutationObserverInit::new();
    config.set_child_list(true);
    config.set_subtree(true);
    config.set_attributes(true);

    // 5. 开始观察。
    observer_handle.observer.observe_with_options(&target, &config)?;
    console::log_1(&format!("Now observing element with id '{}'...", element_id).into());

    // 6. 使用开发者封装的 sleep 函数等待。
    sleep(duration_ms).await;
    console::log_1(&"Observation time finished.".into());

    // 7. 函数结束。
    //    此时 `observer_handle` 将离开作用域，它的 `drop` 方法会被自动调用，
    //    从而执行 `disconnect()` 并释放所有资源。无需手动清理！

    Ok(())
}
```

在 JavaScript 中调用：

```javascript
import init, { observe_dom_changes } from './pkg/your_pkg_name.js';

const run = async () => {
    await init();
    observe_dom_changes('my_element', 5000);
    // 控制台会输出：
    // Now observing element with id 'my-element'...
    const $myBtn = document.getElementById('my_button');
    $myBtn.addEventListener('click', () => {
        const $myElement = document.getElementById('my_element');
        $myBtn.innerHTML = 'my_element content changed';
        // DOM changes detected!
        //  - Mutation: { type: "childList", target: [object Element], addedNodes: [object Text], removedNodes: [object Text] }
    })
};

run();
```

## 三、异步操作：处理 JavaScript Promise

在 JavaScript 中，`Promise` 被广泛用于处理异步操作，而在 Rust 的 WebAssembly 开发中，`wasm-bindgen-futures` 库提供了处理 JavaScript 的 `Promise` 的能力，使得开发者可以在 Rust 中像处理本地 `Future` 一样处理 `Promise`。

### 直接操作`Promise`

```rust
use js_sys::Promise;
use wasm_bindgen::{prelude::wasm_bindgen};
use wasm_bindgen_futures::JsFuture;
use web_sys::{console};

#[wasm_bindgen()]
pub fn call_promise() {
    let promise = Promise::new(&mut |resolve: js_sys::Function, _reject: js_sys::Function| {
        resolve.call0(&"JavaScript Promise resolved!".into()).unwrap();
    });

    let future = JsFuture::from(promise);

    wasm_bindgen_futures::spawn_local(async move {
        let result = future.await.unwrap();
        console::log_1(&format!("Promise resolved with: {:?}", result).into());
    });
}
```

### 使用 `Fetch` API 进行网络请求

在 Web 开发中，网络请求是获取数据和与后端服务交互的重要手段。

`web-sys` 提供了对 `Fetch` API 的支持，让开发者可以在 Rust 中进行网络请求。

以下是一个示例，展示了如何从一个 API 获取数据：

首先，确保 `Cargo.toml` 配置正确：

```toml
[dependencies]
wasm-bindgen-futures = "0.4.50"

[dependencies.web-sys]
version = "0.3.4"
features = [
  'Request', 'RequestInit', 'Response', 'Window', 'fetch', 'RequestMode'
]
```

然后，在 Rust 代码中使用`fetch` API 发起网络请求：

```rust
use wasm_bindgen::{prelude::wasm_bindgen, JsValue};
use wasm_bindgen_futures::JsFuture;
use web_sys::{Request, RequestInit, RequestMode, Response};

#[wasm_bindgen]
pub async fn fetch_json(url: String) -> Result<JsValue, JsValue> {
    // 1. 创建请求
    let opts = RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(RequestMode::Cors);
    let request = Request::new_with_str_and_init(&url, &opts)?;

    // 2. 发起 fetch，它返回一个 Promise
    let window = web_sys::window().unwrap();
    let resp_promise = window.fetch_with_request(&request);

    // 3. 将 Promise 转换为 Future 并 await
    let resp_value = JsFuture::from(resp_promise).await?;
    let resp: Response = resp_value.into();

    // 4. response.json() 同样返回 Promise
    let json_promise = resp.json()?;
    
    // 5.再次 await 获取最终的 JSON 数据 (JsValue)
    let json_data = JsFuture::from(json_promise).await?;

    Ok(json_data)
}
```

## 四、操作 `Canvas` 进行绘图

在 Web 开发中，`Canvas`是一个强大的绘图工具，它允许开发者使用 JavaScript 在网页上绘制各种图形、图像和动画。

`web-sys` 提供了对 `Canvas` API 的支持，通过`web-sys`库可以很方便地获取`Canvas`的 2D 绘图上下文，让开发者可以在 Rust 中进行绘图操作。

以下是一个使用 `Canvas` 绘图的示例，展示了如何在`Canvas`上绘制一个曼德勃罗集图形：

```rust
use wasm_bindgen::{prelude::wasm_bindgen, JsCast};
use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement};

// 曼德勃罗集迭代计算
fn mandelbrot(c_re: f64, c_im: f64, max_iter: u32) -> u32 {
    let mut z_re = 0.0;
    let mut z_im = 0.0;
    
    for i in 0..max_iter {
        // 计算z^2 + c，其中z = z_re + z_im*i，c = c_re + c_im*i
        let z_re_squared = z_re * z_re;
        let z_im_squared = z_im * z_im;
        
        // 如果z的模长平方超过4，说明该点不在曼德勃罗集中
        if z_re_squared + z_im_squared > 4.0 {
            return i;
        }
        
        // 计算下一次迭代的z值
        let z_im_new = 2.0 * z_re * z_im + c_im;
        z_re = z_re_squared - z_im_squared + c_re;
        z_im = z_im_new;
    }
    
    // 如果达到最大迭代次数仍未溢出，认为该点在曼德勃罗集中
    max_iter
}

// 将迭代次数转换为颜色
fn get_color(iterations: u32, max_iter: u32) -> String {
    if iterations == max_iter {
        // 曼德勃罗集内部，使用黑色
        return "#000000".to_string();
    }
    
    // 简单的颜色映射，将迭代次数转换为RGB颜色
    let t = iterations as f64 / max_iter as f64;
    let r = (9.0 * (1.0 - t) * t * t * t * 255.0) as u8;
    let g = (15.0 * (1.0 - t) * (1.0 - t) * t * t * 255.0) as u8;
    let b = (8.5 * (1.0 - t) * (1.0 - t) * (1.0 - t) * t * 255.0) as u8;
    
    format!("#{:02x}{:02x}{:02x}", r, g, b)
}

#[wasm_bindgen]
pub fn draw_fractal(canvas_id: String) {
    // 获取文档和Canvas元素
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas_element = document.get_element_by_id(&canvas_id)
        .expect("Canvas element not found");
    
    let canvas: HtmlCanvasElement = canvas_element
        .dyn_into::<HtmlCanvasElement>()
        .expect("Element is not a canvas");
    
    // 设置Canvas尺寸
    let width = 800.0;
    let height = 600.0;
    canvas.set_width(width as u32);
    canvas.set_height(height as u32);
    
    // 获取2D渲染上下文
    let ctx: CanvasRenderingContext2d = canvas
        .get_context("2d")
        .expect("Failed to get 2d context")
        .expect("2d context is not available")
        .dyn_into()
        .expect("Failed to convert to CanvasRenderingContext2d");
    
    // 曼德勃罗集参数
    let max_iter = 1000;
    let x_min = -2.0;
    let x_max = 1.0;
    let y_min = -1.0;
    let y_max = 1.0;
    
    // 计算每个像素对应的复数平面坐标
    let x_range = x_max - x_min;
    let y_range = y_max - y_min;
    
    // 绘制曼德勃罗集 - 使用逐像素绘制
    for x in 0..(width as u32) {
        for y in 0..(height as u32) {
            // 将像素坐标映射到复数平面
            let c_re = x_min + (x as f64 / width) * x_range;
            let c_im = y_min + (y as f64 / height) * y_range;
            
            // 计算该点的迭代次数
            let iterations = mandelbrot(c_re, c_im, max_iter);
            
            // 设置颜色并绘制像素
            ctx.set_fill_style(&get_color(iterations, max_iter).into());
            ctx.fill_rect(x as f64, y as f64, 1.0, 1.0);
        }
    }
}
```

在 JavaScript 中调用`draw_fractal`函数后，效果图如下：


![WX20250809-103014@2x.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/9fddc18c606b460784b8d8a4e53a0257~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1754812158&x-orig-sign=hnJsQsF%2BSYDBhM964g75ZnlNZOQ%3D)
## 五、与 localStorage 或 sessionStorage 交互

在 Web 开发中，`localStorage`和`sessionStorage`是用于在客户端本地存储数据的重要机制。

`localStorage`存储的数据是持久化的，只要不手动清除或浏览器卸载，数据就会一直存在；而`sessionStorage`存储的数据仅在当前会话（即浏览器窗口打开期间）有效，当窗口关闭时，数据会被清除。

借助`web-sys`库，开发者可以方便地与`localStorage`和`sessionStorage`进行交互。

通过 `web-sys` 库, 使用 `localStorage` 和`sessionStorage` 进行存储数据、读取数据及删除数据的代码示例如下：

```rust
use web_sys::window;
use wasm_bindgen::prelude::*;

/// 向localStorage存储数据
#[wasm_bindgen]
pub fn set_local_storage(key: &str, value: &str) -> Result<(), JsValue> {
    // 获取window对象，若不存在则返回错误
    let window = window().ok_or_else(|| JsValue::from_str("Window object not available"))?;
    // 获取localStorage，若不存在则返回错误
    let storage = window.local_storage()?.ok_or_else(|| JsValue::from_str("localStorage is not available"))?;
    
    // 存储数据
    storage.set_item(key, value)?;
    Ok(())
}

/// 从localStorage读取数据
#[wasm_bindgen]
pub fn get_local_storage(key: &str) -> Result<Option<String>, JsValue> {
    // 获取window对象，若不存在则返回错误
    let window = window().ok_or_else(|| JsValue::from_str("Window object not available"))?;
    // 获取localStorage，若不存在则返回错误
    let storage = window.local_storage()?.ok_or_else(|| JsValue::from_str("localStorage is not available"))?;
    
    // 读取数据，返回Option<String>
    Ok(storage.get_item(key).unwrap())
}

/// 从localStorage删除数据
#[wasm_bindgen]
pub fn remove_local_storage(key: &str) -> Result<(), JsValue> {
    // 获取window对象，若不存在则返回错误
    let window = window().ok_or_else(|| JsValue::from_str("Window object not available"))?;
    // 获取localStorage，若不存在则返回错误
    let storage = window.local_storage()?.ok_or_else(|| JsValue::from_str("localStorage is not available"))?;
    storage.remove_item(key)?; 
    Ok(())
}

/// 向sessionStorage存储数据
#[wasm_bindgen]
pub fn set_session_storage(key: &str, value: &str) -> Result<(), JsValue> {
    // 获取window对象，若不存在则返回错误
    let window = window().ok_or_else(|| JsValue::from_str("Window object not available"))?;
    // 获取sessionStorage，若不存在则返回错误
    let storage = window.session_storage()?.ok_or_else(|| JsValue::from_str("sessionStorage is not available"))?;
    
    // 存储数据
    storage.set_item(key, value)?;
    Ok(())
}

/// 从sessionStorage读取数据
#[wasm_bindgen]
pub fn get_session_storage(key: &str) -> Result<Option<String>, JsValue> {
    // 获取window对象，若不存在则返回错误
    let window = window().ok_or_else(|| JsValue::from_str("Window object not available"))?;
    // 获取sessionStorage，若不存在则返回错误
    let storage = window.session_storage()?.ok_or_else(|| JsValue::from_str("sessionStorage is not available"))?;
    
    // 读取数据，返回Option<String>
    Ok(storage.get_item(key).unwrap())
}
    
/// 删除sessionStorage中的数据
#[wasm_bindgen]
pub fn remove_session_storage(key: &str) -> Result<(), JsValue> {
    // 获取window对象，若不存在则返回错误
    let window = window().ok_or_else(|| JsValue::from_str("Window object not available"))?;
    // 获取sessionStorage，若不存在则返回错误
    let storage = window.session_storage()?.ok_or_else(|| JsValue::from_str("sessionStorage is not available"))?;
    
    // 删除数据
    storage.remove_item(key)?;
    Ok(())
}
    
```

在JavaScript中调用这些函数：

```javascript
import init, { set_local_storage, get_local_storage, set_session_storage, get_session_storage } from './pkg/your_pkg_name.js';

const run = async () => {
    await init();
    set_local_storage('my_local_key', 'my_local_key_value');
    const value = get_local_storage('my_local_key');
    console.log(value);
    set_session_storage('my_session_key', 'my_session_key_value');
    const value1 = get_session_storage('my_session_key');
    console.log(value1);
};

run();
```

## 六、定时器控制

在 Web 开发中，定时器允许开发者在指定的时间间隔后执行代码，或者延迟一段时间后执行代码。

借助`web-sys`库，开发者可以方便地使用`setTimeout`和`setInterval`这两个定时器函数。

并且可以使用`clearTimeout`和`clearInterval`方法来清除已设置的定时器，以避免内存泄漏和不必要的资源消耗。

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::window;
use js_sys::{Function, Array};

#[wasm_bindgen]
pub struct Timer {
    closure: Option<Closure<dyn FnMut()>>, // 使用Option以便安全取出
    id: Option<i32>,                       // 使用Option跟踪状态
}

#[wasm_bindgen]
impl Timer {
    #[wasm_bindgen(constructor)]
    pub fn new(callback: &Function, interval: i32) -> Result<Timer, JsValue> {
        // 确保interval为非负值
        if interval < 0 {
            return Err(JsValue::from_str("Interval cannot be negative"));
        }
        
        // 获取window对象，妥善处理可能的None
        let window = window()
            .ok_or_else(|| JsValue::from_str("Window object not available"))?;
        
        // 创建闭包，捕获回调函数的克隆（而非引用）以避免生命周期问题
        let callback = callback.clone();
        let closure = Closure::wrap(Box::new(move || {
            let _ = callback.call0(&JsValue::NULL);
        }) as Box<dyn FnMut()>);
        
        // 设置定时器，妥善处理可能的错误
        let id = window
            .set_interval_with_callback_and_timeout_and_arguments(
                closure.as_ref().unchecked_ref(),
                interval,
                &Array::new(),
            )
            .map_err(|e| JsValue::from_str(&format!("Failed to create interval: {:?}", e)))?;
        
        Ok(Timer {
            closure: Some(closure),
            id: Some(id),
        })
    }
    
    /// 取消定时器并释放资源
    #[wasm_bindgen]
    pub fn cancel(&mut self) {
        // 清除定时器（如果存在）
        if let Some(id) = self.id.take() {
            // 忽略清除时的错误，因为此时窗口可能已关闭
            let _ = window().and_then(|w| Some(w.clear_interval_with_handle(id)));
        }
        
        // 取出并释放闭包
        if let Some(closure) = self.closure.take() {
            // 转换为JsValue后释放，确保JavaScript可以回收内存
            closure.into_js_value();
        }
    }
    
    /// 检查定时器是否仍在运行
    #[wasm_bindgen]
    pub fn is_active(&self) -> bool {
        self.id.is_some() && self.closure.is_some()
    }
}

// 实现Drop trait确保资源释放
impl Drop for Timer {
    fn drop(&mut self) {
        // 如果用户忘记调用cancel()，在这里自动清理
        if self.is_active() {
            self.cancel();
        }
    }
}

```

在 JavaScript 中 创建 `Timer` 实例：

```javascript
import init, { Timer } from './pkg/your_pkg_name.js';

const run = async () => {
    await init();
   let count = 0;
    const timer = new Timer(
        () => {
            count += 1;
            console.log('wasm call count: ', count);
        },
        1000,
    );
    // 10秒后取消定时器
    setTimeout(() => {
        console.log('js cancel timer');
        timer.cancel();
    }, 10000); 
    // 输出如下：
    // wasm call count:  1
    // wasm call count:  2
    // ...
    // wasm call count:  9 
    // wasm call count:  10
    // js cancel timer
};

run();
```

## 七、总结

通过 `web-sys`，Rust 获得了与浏览器深度交互的能力。从事件处理到 Canvas 操作，从异步 Promise 到本地存储，Rust 在浏览器环境中展现出前所未有的强大能力。

**关键技能：**

*   **高级事件处理**：获取事件详情，动态增删事件监听。
*   **异步操作**：使用 `wasm-bindgen-futures` 征服 `Promise` 和 `fetch`。
*   **Canvas 绘图**：释放 Rust 的计算能力进行图形创作。
*   **Web API 实践**：熟练运用 `Storage` 的 `localStorage` 和 `sessionStorage`进行数据存取。
*   **定时器控制**：使用 `setTimeout` 和 `setInterval` 进行精确的时间控制。
