# 用 Rust 构建下一代机器人应用：dora-rs 入门与实战

## 一、什么是 dora-rs？

**dora-rs**（Dataflow-Oriented Robotic Architecture）是一个面向 AI 时代的具身智能机器人中间件框架。它将机器人应用抽象为**节点（Node）** 和 **数据流（Dataflow）** 组成的有向无环图，这种范式从根本上促进了系统的模块化、可配置性和可扩展性。

简单来说，dora-rs 是一个帮你搭建复杂机器人程序的 **“乐高积木”** 框架。

**核心设计思想**：

*   **节点化**：每个功能模块（视觉、决策、控制）都是独立节点
*   **声明式数据流**：通过 `YAML` 定义节点连接关系，无需关心底层通信
*   **零拷贝传输**：基于 `Apache Arrow` 和 `共享内存`，实现跨进程数据零拷贝
*   **多语言支持**：Rust 节点高性能，Python 节点方便调用 AI 生态


![dora01.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/676b18949e7a45a498e444634f2240d0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1765378511&x-orig-sign=hMiyRNd5DlKZ0ZRw10MOcyhJfwo%3D)

## 二、为什么选择 dora-rs ？

dora-rs 是一款高性能机器人框架，它的设计哲学是 “零成本抽象”，通过三个核心机制解决了延迟问题。

#### 1. Dataflow 架构：从“嘈杂聊天室”到“精密流水线”

dora 模式 (Dataflow) 就像一条精密的工厂流水线，数据就是原材料：

*   **Node (节点)**： 流水线上的工人（负责推理、控制等独立任务）。

*   **Edge (边)**： 传送带（定义数据流动的方向）。

**机制**： 节点之间通过 `YAML` 配置文件声明连接关系，系统自动实现数据驱动执行，天然支持并行性。

#### 2. Zero-Copy（零拷贝）：打破内存的高墙 🚀

传统 `IPC` 需要多次数据拷贝，`dora-rs` 通过共享内存 + `Arrow 格式`，让多个进程直接访问同一块内存，避免拷贝开销。

其中 `Arrow` 格式是一种列式内存格式，它提供了一种语言无关的标准方式来表示数据。`dora-rs` 要求数据必须以 `Arrow` 格式传递，这样无论数据流向 `Rust` 节点还是 `Python` 节点，都能保证内存布局相同，从而实现零拷贝。

*   **传统流程**： 内存读取 -> 序列化 -> Socket发送 -> 反序列化 -> 写入内存。（至少 4 次数据拷贝，消耗大量 CPU）。

*   **dora 流程**： 数据只在共享内存中写入一次。节点间传递的只是这块内存的 **“引用（指针）”**。

#### 3. **Rust 原生实现**

*   **内存安全**： Rust 的所有权（Ownership）系统在编译期就解决了内存安全问题，杜绝了传统 `C++` 机器人中间件中常见的野指针和段错误。

*   **极致性能**： Rust **无运行时开销（Zero-cost Abstraction）** 的特性，配合零拷贝，使性能逼近硬件极限。

***

## 三、适用场景对比

| 特性        | dora-rs           | ROS2       | 适用建议                  |
| --------- | ----------------- | ---------- | --------------------- |
| **延迟**    | 1-2ms             | 15-20ms    | dora-rs 更擅长 高带宽、低延迟场景 |
| **内存安全**  | 编译期保证             | 运行时检查      | Rust 避免野指针和数据竞争       |
| **生态成熟度** | 早期发展阶段            | 非常成熟       | ROS2 适合快速复用现有算法       |
| **开发门槛**  | 需掌握 Rust          | Python 友好  | Python 开发者上手更快        |
| **典型场景**  | 工业控制、边缘 AI、高性能机械臂 | 通用机器人、科研验证 | 根据项目需求选择              |

**建议**：`dora-rs` 适合对**实时性要求极高**的场景，或需要处理多路高带宽数据流（如多摄像头、激光雷达）的系统。如果项目需要复用大量现有 `ROS2` 包，可混合使用：`dora-rs` 处理高性能数据流，通过桥接复用 `ROS2` 生态。

***

## 四、Mac 环境搭建（Apple Silicon）

### 安装步骤

```bash
# 1. 安装 Rust（如未安装）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# 2. 安装 dora-cli（推荐一键脚本）
cargo install dora-cli

# 3. 验证安装
dora --version  # 应显示 v0.3.13

```

其他环境可参考 [dora-rs 安装文档](https://dora-rs.ai/docs/guides/Installation/installing)

***

## 五、实战：Rust 实现摄像头读取与显示

### 项目结构

```bash
dora-webcam-rust/
├── Cargo.toml        # Workspace配置
├── dataflow.yml      # 数据流定义
├── webcam/       # 摄像头节点
│   ├── Cargo.toml
│   └── src/main.rs
└── viewer/         # 显示节点
    ├── Cargo.toml
    └── src/main.rs
```

### 1. 创建项目

```bash
# 创建项目, rust语言
dora new --lang rust dora-webcam-rust

# 删除默认创建的节点
rm -rf listener_1 talker_1 talker_2

# 创建自定义节点 - 摄像头节点
dora new --kind node  --lang rust webcam
# Created new Rust custom node `webcam` at ./webcam

# 创建自定义节点 - 显示节点
dora new --kind node  --lang rust viewer
# Created new Rust custom node `viewer` at ./viewer

```

### 2. 安装 opencv

获取摄像头数据，需要安装 opencv 及对应的rust绑定

```bash
brew install opencv # macos 安装opencv 4.12.x
```

### 3. 编写摄像头节点

**`webcam/Cargo.toml`**

```toml
[package]
name = "webcam"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
dora-node-api = "0.3.13"
opencv = { version = "0.97.2", features = ["videoio", "imgcodecs"] }
anyhow = "1.0"
```

**`webcam/src/main.rs`**

```rust
use dora_node_api::{DoraNode, Event, dora_core::config::DataId, arrow::array::UInt8Array};
use std::error::Error;
use std::thread;
use std::time::Duration;
use anyhow::Context;
use opencv::{
    core::{Vector}, imgcodecs, prelude::*, videoio::{self, VideoCapture}
};

const CAMERA_INDEX: i32 = 0; // 默认使用第一个摄像头

fn main() -> Result<(), Box<dyn Error>> {
    let (mut node, mut events) = DoraNode::init_from_env()?;
    let mut camera = VideoCapture::new(CAMERA_INDEX, videoio::CAP_ANY)
        .context("Failed to create video capture")?;
    let output = DataId::from("frame".to_owned());
    // 尝试打开摄像头
    if !VideoCapture::is_opened(&camera).context("Failed to check if camera is open")? {
        // 在 Mac M1 上，有时需要延迟以确保摄像头初始化完成
        thread::sleep(Duration::from_millis(500));
        if !VideoCapture::is_opened(&camera).context("Camera still not open after delay")? {
            return Err("Could not open camera 0 or check its status.".into());
        }
    }

    // 尝试设置分辨率 (可选，可以提高性能或稳定性)
    // camera.set(videoio::CAP_PROP_FRAME_WIDTH, 640.0)?;
    // camera.set(videoio::CAP_PROP_FRAME_HEIGHT, 480.0)?;

    while let Some(event) = events.recv() {
        // println!("Received event: {:?}", event);
        match event {
            Event::Input {
                id,
                metadata,
                data: _,
            } => match id.as_str() {
                "tick" => {
                    let mut frame = Mat::default();
                    
                    // 读取帧
                    if camera
                        .read(&mut frame)
                        .context("Failed to read frame from camera")?
                    {
                        if frame.size().context("Failed to get frame size")?.width > 0 {
                            // 将帧编码为 JPEG 格式的字节向量
                            let mut buffer = Vector::new();
                            imgcodecs::imencode(".jpg", &frame, &mut buffer, &Vector::new())
                                .context("Failed to encode frame to JPEG")?;

                            // 发送原始帧数据
                            let std_buffer: Vec<u8> = buffer.into_iter().collect();

                            // 3. 再转为 Arrow 数组
                            let arrow_array = UInt8Array::from(std_buffer);
                            node.send_output(
                                output.clone(),
                                metadata.parameters,
                                arrow_array,
                            )?;
                        }
                    }
                }
                other => eprintln!("Received input `{other}`"),
            },
            _ => {}
        }
    }

    Ok(())
}

```

### 3. 编写显示节点（接收处理）

**`viewer/Cargo.toml`**

```toml
[package]
name = "viewer"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
dora-node-api = "0.3.13"
opencv = { version = "0.97.2", features = ["highgui", "imgcodecs"] }
anyhow = "1.0"
```

**`viewer/src/main.rs`**

```rust
use anyhow::Context;
use dora_node_api::{arrow::array::UInt8Array, DoraNode, Event};
use opencv::{core::Vector, highgui, imgcodecs, prelude::*};
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let (mut _node, mut events) = DoraNode::init_from_env()?;
    // 创建一个用于显示的窗口
    highgui::named_window("Dora Webcam Viewer (Rust)", highgui::WINDOW_NORMAL)
        .context("Failed to create highgui window")?;
    println!("Viewer operator initialized.");
    while let Some(event) = events.recv() {
        match event {
            Event::Input { id, metadata: _, data } => match id.as_str() {
                "frame" => {
                    // 将接收到的字节数据转换为 OpenCV Vector
                    // 1. 将 Arrow trait 对象强转为具体的 UInt8Array
                    let uint8_array = data
                        .as_any()
                        .downcast_ref::<UInt8Array>()
                        .context("Arrow data is not UInt8Array (expected byte array)")?;

                    // 2. 提取 UInt8Array 的字节切片
                    let byte_slice = uint8_array.values(); // 返回 &[u8]

                    // 3. 转换为 OpenCV Vector<u8>（from_slice 接收 &[u8]）
                    let buffer = Vector::from_slice(byte_slice);

                    // 解码 JPEG 数据成 Mat
                    let frame = imgcodecs::imdecode(&buffer, imgcodecs::IMREAD_COLOR)
                        .context("Failed to decode image from buffer")?;

                    if frame
                        .size()
                        .context("Failed to get decoded frame size")?
                        .width
                        > 0
                    {
                        // 显示图像
                        highgui::imshow("Dora Webcam Viewer (Rust)", &frame)
                            .context("Failed to imshow frame")?;
                        // 必须调用 wait_key 来处理 GUI 事件
                        highgui::wait_key(1).context("Failed to wait_key")?;
                    }
                }
                other => eprintln!("Received input `{other}`"),
            },
            _ => {}
        }
    }

    Ok(())
}

```

### 4. 定义数据流

**`dataflow.yml`**

```yaml
nodes:
  - id: webcam
    build: cargo build -p webcam
    path: target/debug/webcam
    inputs:
      tick: dora/timer/millis/100
    outputs:
      - frame

  - id: viewer
    build: cargo build -p viewer
    path: target/debug/viewer
    inputs:
      frame: webcam/frame

```

### 5. 运行数据流

```bash
# 构建数据流
dora build dataflow.yml

# 启动守护进程
dora up

# 运行数据流（查看日志）
dora start dataflow.yml
```

**预期输出**：

```bash
attaching to dataflow (use `--detach` to run in background)
INFO   dora daemon    finished building nodes, spawning...
viewer: INFO   spawner    spawning `/Users/gustaf/practice/rust/dora-webcam-rust/target/debug/viewer` in `/Users/gustaf/practice/rust/dora-webcam-rust`
webcam: INFO   spawner    spawning `/Users/gustaf/practice/rust/dora-webcam-rust/target/debug/webcam` in `/Users/gustaf/practice/rust/dora-webcam-rust`
webcam: INFO   daemon    node is ready
viewer: INFO   daemon    node is ready
INFO   daemon    all nodes are ready, starting dataflow
viewer: stdout    Viewer operator initialized.
```

**核心要点**：数据在两个独立进程间传输，但**没有发生内存拷贝**，仅传递了引用。

**运行效果** 

![result.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/dc6f07dd7b16495d98b2bc10c3040552~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1765378702&x-orig-sign=uWeDida80hRciZnBQU4NGP5aos4%3D)

**GitHub仓库**：<https://github.com/Doomking/rust-zhixingshe-examples/tree/main/dora/dora-webacm-rust>

***

## 六、学习资源与进阶路径

### 📚 官方资源

*   **文档**：<https://dora-rs.ai/docs/>
*   **GitHub**：<https://github.com/dora-rs/dora>
*   **示例**：[examples目录](https://github.com/dora-rs/dora/tree/main/examples)

### 💬 社区交流

*   **中文社区**：[doracc.com](https://doracc.com)
*   **Discord**：<https://discord.gg/DXJ6edAtym>

### 🎯 实践建议

1.  **接入摄像头**：用`opencv`读取USB摄像头
2.  **集成AI模型**：添加`YOLO`目标检测节点
3.  **分布式部署**：尝试多机协同
4.  **性能分析**：使用火焰图分析热点

***

## 七、总结

`dora-rs` 为高性能机器人开发提供了新选择：

*   **性能**：零拷贝和低延迟适合实时系统
*   **安全**：Rust内存安全保障系统稳定性
*   **灵活**：多语言支持和声明式配置简化开发

选择技术栈时，建议根据项目需求权衡：`dora-rs` 适合对性能敏感的新项目，`ROS2` 适合复用现有生态的场景。
