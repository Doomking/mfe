# 🤖 终于来了！这个开源机器人开发框架Dora，用Python/Rust让机器人编程像搭积木一样简单！

![cover.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/461edc86a2274dc4947c8621f09eaaf4~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1764937922&x-orig-sign=hVv7oGapxXLAXkIekAtOsCgvw04%3D)
---

### **引言：告别C/C++，迎接“平民化”机器人时代**

如果你曾经接触过机器人编程，一定对那些复杂的 `C/C++` 代码和漫长的调试过程感到头疼。

对于众多机器人爱好者、甚至硕士博士学生来说，投入数周时间深陷底层代码的泥潭，无疑是一种巨大的挑战。

传统的机器人开发，就像是建造一座大厦，你需要掌握每一块砖的烧制方法和钢筋的熔炼技术，才能开始搭建。

然而，一个革命性的 `Rust` 开源项目正在打破这一壁垒——它就是 **Dora (Dataflow Oriented Robotics Architecture)**。`Dora` 的出现，好比为你提供了预制好的模块和简洁的施工图纸，让建造变得高效且人人可为。

由 `Xavier Tao` 和 `Philipp Oppermann` 共同创立的`Dora`，旨在将机器人编程的门槛降至最低，让开发者能够使用更友好、更高效的语言（如 **Python** 或 **Rust**）来控制机器人，推动机器人技术真正走向大众。这不仅能加速研究与开发，更能激发无数创意，让机器人在更多领域发挥价值。

### **一、认识Dora：数据流驱动的机器人“中枢神经”**

Dora本质上是一个全新的**开源中间件框架**和**数据流计算平台**，专为自动驾驶和机器人领域设计。它的核心理念是：**将复杂的机器人任务拆解成高效、并行的数据流。**

与传统的应用程序不同，机器人编程涉及多个需要高效通信的并行循环（例如：摄像头捕捉、AI模型计算、运动控制）。想象一下，机器人就像一个复杂的交响乐团，每个乐手（模块）都在演奏自己的部分，但需要完美地协同才能奏出和谐的乐章。`Dora` 正是那个高效的指挥家和精密的乐谱。

它通过引入以下机制，完美解决了这一痛点：

#### **1. 数据流架构：节点 (Node) 与边 (Edge)**

Dora 采用直观的数据流图来组织机器人应用。

* **节点 (Node):** 代表一个独立运行的并行主循环。它可以是一个传感器驱动程序，负责从激光雷达获取数据；也可以是一个复杂的SLAM算法，实时构建环境地图；抑或是一个AI推理模型，识别图像中的物体。每个节点都像一个独立的微服务，专注于完成自己的任务。
* **边 (Edge):** 相当于通信主题（Topic）。节点之间通过发布/订阅模式进行数据交换。例如，一个摄像头节点发布图像数据，而一个视觉处理节点订阅这些图像数据进行分析。
* **配置简化：** 整个数据流图的构建只需通过一个 **YAML 文件**进行声明，就像使用Docker Compose或Kubernetes管理容器服务一样，极大地简化了系统集成。这意味着开发者可以专注于业务逻辑，而不是繁琐的配置和底层通信。

![Gemini_Generated_Image_xy9lgsxy9lgsxy9l.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/360d9eeca0a84f9c95aea2c6a5521cd9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1764937843&x-orig-sign=L83c%2BEIuuX911F6e3mpnNT9mkt8%3D)

#### **2. 通信革命：基于 Apache Arrow 的“零拷贝”**

这是Dora实现高性能的关键所在，也是它在众多机器人框架中脱颖而出的重要原因。

在传统机器人系统中，不同语言和应用之间传递大数据（如图像、点云、传感器阵列数据）时，往往需要进行昂贵的序列化和反序列化操作，以及内存拷贝。这不仅消耗大量的CPU资源，还可能带来 **20毫秒甚至更高的延迟**，这在实时性要求极高的机器人应用中是致命的。

Dora 采用了 **Apache Arrow** 统一存储布局来解决这一难题：

* **统一格式：** Arrow 为不同语言（C/C++、Rust、Python）定义了一个**统一的内存存储格式**。无论数据源自何种语言编写的模块，它都能以相同的高效格式存在于内存中。
* **零拷贝 (Zero-Copy)：** 这使得所有消息可以在不经过额外拷贝的情况下，高效地在不同节点之间传递。想象一下，数据不再是像快递包裹一样需要层层包装和拆卸，而是直接在内存中共享“地址”，从而**大幅降低了通信延迟**，实现近乎实时的信息交互。这对于需要处理高清视频流、高密度点云等大数据的机器人应用至关重要。


![Gemini_Generated_Image_j4yjtrj4yjtrj4yj.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/990c9cf2631d43d4a96d5efa63789573~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1764937871&x-orig-sign=5jAIWalKTFPgmsvwbLPJHnlZOiE%3D)

### **三、多模型集成与效率飞跃：从代码到指令**

`Dora` 不仅解决了底层通信效率问题，更致力于整合最新的AI能力，让机器人变得更智能、更具交互性。

#### **1. 轻松集成SOTA模型，接收非结构化数据**

得益于其卓越的性能和模块化设计，`Dora` 可以轻松集成各种SOTA（State-of-the-Art）模型，为机器人赋予强大的感知和理解能力：

* **OpenAI Whisper：** 集成后，机器人可以准确地将人类的语音指令转化为文本，理解你的意图。
* **VLM (Vision-Language Model) 模型：** 例如CLIP、BLIP等，使机器人能够从图像中生成文本描述，理解视觉场景，甚至根据视觉内容回答问题。
* **Hugging Face Parler：** 实现文本转语音（Text-to-Speech），让机器人能够用自然流畅的语言与人交流。

这意味着机器人应用程序现在可以接受**非结构化数据**（如语音、图像、手势）作为输入。你可以通过简单的**语音指令**（如“去办公室拿那本书”、“把这个递给我”）或**手势**来控制机器人的行为，极大地丰富了人机交互的可能性，使机器人不再是冷冰冰的机器，而是更像一个智能助手。


![Gemini_Generated_Image_d9hs7vd9hs7vd9hs.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/e788d569a89243c0b8ea3e3e25ff3b35~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1764937906&x-orig-sign=8ZAS6gXx0GGHjuw0aabJNv4e2UI%3D)

#### **2. 热加载 (Hot Reloading)：实时调试的极致体验**

在传统的嵌入式或机器人开发中，每次修改一行代码，可能都需要漫长的编译、烧录、重启系统、初始化传感器等步骤，耗时费力，严重影响开发效率。

`Dora` 借鉴于Web开发领域成熟的**热加载**功能，为机器人开发带来了革命性的效率提升：

* **实时更改，无需重启：** 开发者可以对代码进行实时修改，无需关闭机器人、重启各种传感器和应用。这意味着你可以像调试网页一样调试机器人程序，即时看到修改后的效果。
* **故障安全机制：** `Dora` 内置的故障安全机制确保了在热加载过程中系统的稳定性和安全性。
* **与LLMs结合的策略编码：** 热加载的特性使其能更好地与**大语言模型 (LLMs)** 结合，实现**策略编码（Policy Coding）**。LLMs 可以根据用户的自然语言指令，生成或修改机器人行为策略。`Dora` 能够将这些策略更新实时应用到运行中的机器人上，将视觉或语音指令实时映射到机器人的执行动作，极大地加速了机器人的学习和适应能力。

### **四、实战案例：50行代码，机械臂听你指挥**

`Dora` 团队在机械臂控制领域取得了令人瞩目的进展，充分展示了其框架的强大能力和开发效率。

* **极简代码实现复杂控制：** 过去，控制机械臂完成“拿起”动作可能需要数百行甚至上千行复杂的 `C++` 代码，涉及运动学、动力学、路径规划等多个模块。然而，`Dora` 团队仅用 **50行Rust代码**，便在一周内实现了通过语音指令控制机械臂执行“从桌边拿起一块巧克力”的复杂任务。这不仅是代码量的巨大缩减，更是开发周期的大幅缩短。
* **性能突破：** 与SOTA机械臂（如Aloha）的遥操作系统相比，`Dora` 不仅能够实现同样的功能，还将其频率提高了 **10倍**（从50Hz到500Hz），这意味着更流畅、更精准的实时控制。同时，将两个机械臂之间的通信延迟降至令人惊叹的 **2毫秒**，这在需要高精度协同操作的任务中至关重要。
* **普适性：** Dora-Robotic-Arm 支持**跨平台**（Linux、MacOS、Windows）和**多语言**（Rust、C、C++、Python），彻底摆脱了复杂的安装依赖，使用 `cargo` 或 `pip` 即可轻松部署。这种灵活性使得更多的开发者能够轻松上手，无论他们熟悉哪种语言。

### **五、社区与愿景：开源精神驱动未来**

`Dora` 的出现，证明了机器人技术不再是少数大型实验室和企业的“专属领地”。创始人陶海轩（Xavier Tao）和团队致力于将最好的机器人技术理念，从闭源社区带向开源，确保每个人都能更容易地开始机器人开发。

他们坚信，通过开放协作，能够加速机器人技术的创新，让机器人更好地服务于人类生活。`Dora` 不仅仅是一个技术框架，更是一种开放、普惠的理念，它邀请所有对机器人充满热情的人，共同构建更智能、更易用的机器人未来。

如果你渴望用简单、高效的方式拥抱机器人和AI的结合，`Dora`无疑是你最值得关注和尝试的开源框架。它提供了一个强大而灵活的平台，让你的创意能够迅速转化为现实。

**项目地址（GitHub）：**
[https://github.com/dora-rs/dora](https://github.com/dora-rs/dora)

---
*欢迎分享你的看法，你认为Dora能否成为下一代机器人开发的标准？你对未来的“平民化”机器人有何期待？*