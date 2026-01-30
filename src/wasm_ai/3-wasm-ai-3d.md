# Rust + Wasm + AI（三）：浏览器里的二次元伴侣——LLM+3D数字人集成实战

**你的AI伴侣，不该只活在文本框里**

在本系列中，我们已经完成了从情感分析到多轮对话的跨越。但无论 AI 多聪明，交互界面始终是一个冷冰冰的文本框。

今天，我们来弄个不一样的：**把你的AI变成3D二次元妹子，让她在浏览器里对你笑、跟你聊、给你讲笑话还会捂嘴偷笑那种。**

最重要的是，全程不依赖任何服务器，本地运行 Qwen 大语言模型，让你的小姐姐只活在你一个人的电脑里。

## 一、先看效果：你的3D智能伴侣长啥样？

![output13.jpg](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/cfcfd6c684f4421997bf329607f5e8e0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1769840209&x-orig-sign=0jBB9bTndiIQhs5Nn%2BW04u1sSxU%3D)

你打开网页，一个蓝发双马尾的少女站在草地上，眼睛一眨一眨。你在聊天框输入："讲个程序员笑话"。她**右手抬起，调皮地捂住嘴，眼睛眯成月牙，眉毛上扬**，然后开始说话（真的有声音！），嘴巴开合跟语音完美同步。说完后放下手，恢复自然站立，还轻轻晃了晃身子，像在说"嘿嘿，好笑吧"。

**交互流程：**

- **感知层**：用户输入 Prompt。
- **大脑层**：Worker 线程中的 Rust 引擎（Wasm）进行流式推理，实时吐出 Token。
- **表达层**：Web Speech API 将文字转为声音。
- **形态层**：根据语音，驱动 3D 模型的 BlendShape（口型）与骨骼动画。

上面的效果是不是挺酷？别急，完整的代码实现和模型资源我都放在文末了，跟着教程一步步来，你也能拥有一个专属的3D AI伴侣。

## 二、LLM大脑：Qwen 0.5B轻量级部署

### 2.1 为什么选Qwen 0.5B？

在浏览器跑大模型，最怕的就是**内存爆炸**。我们来看看 LLM 初始化代码：

```rust
// lib.rs - 模型初始化
pub async fn init(
    weights_data: Vec<u8>,
    tokenizer_data: Vec<u8>,
    config_data: Vec<u8>,
) -> Result<LLMEngine, JsValue> {
    // 关键决策：先用CPU推理，稳
    let device = Device::Cpu;

    let tokenizer = Tokenizer::from_bytes(&tokenizer_data)?;
    let config: QwenConfig = serde_json::from_slice(&config_data)?;

    // 从SafeTensors加载权重，DType::F32保证兼容性
    let vb = VarBuilder::from_buffered_safetensors(
        weights_data,
        DType::F32,
        &device
    )?;

    let model = QwenModel::new(&config, vb)?;
    // ...返回引擎
}
```

**选型逻辑：**

- **内存友好**：0.5B参数+FP32精度≈3GB内存占用，在16GB内存的Windows和Mac上都能流畅运行
- **中文特化**：阿里Qwen对中文环境、日常对话的响应质量比较好
- **Instruct微调**：直接支持对话格式，不用我们折腾Prompt工程

### 2.2 KV Cache：让AI说话不卡顿

为了保证“首字秒回”，我们利用了 Rust 框架 `candle` 中的 **KV Cache（键值缓存）**。

简单说，模型不需要每次都从头读 Prompt，而是记住之前的状态。

```rust
// lib.rs - 核心增量推理逻辑
pub fn step(&mut self) -> Result<String, JsError> {
    // 1. 确定步长：首轮推理全量，后续每步只推 1 个 token
    let context_size = if self.pos == 0 { self.token_ids.len() } else { 1 };
    let start_idx = self.token_ids.len() - context_size;

    let input_tensor = Tensor::new(&self.token_ids[start_idx..], &self.device)?
        .unsqueeze(0)?;

    // 2. 前向传播：传入当前位置 pos，激活 KV Cache 机制
    let logits = self.model.forward(&input_tensor, self.pos)?;
    self.pos += context_size;

    // 3. 重复惩罚：防止 AI 陷入“复读机”死循环
    let mut logits_vec = (logits / 0.7).flatten_all()?.to_vec1::<f32>()?;
    let penalty = 1.1f32;
    for &id in &self.token_ids {
        let idx = id as usize;
        if idx < logits_vec.len() {
            logits_vec[idx] = if logits_vec[idx] > 0.0 { logits_vec[idx] / penalty }
                              else { logits_vec[idx] * penalty };
        }
    }
    // ... 后续 Argmax 采样与解码 ...
}
```

**注：** 第一次推理，模型要处理你完整的输入，比如"讲个程序员笑话"这7个字，它会把中间计算结果（Key/Value矩阵）全部存起来了。当生成第一个Token后，后面每次只需要把新Token送进去，复用缓存，速度明显提升。

这种"先慢后快"的节奏，正好跟人类说话一样——开头思考一下，后面就连珠炮了。

### 2.3 温度与重复惩罚：让AI更像人

```rust
// lib.rs - 采样逻辑
let logits = (logits / 0.7)?;  // temperature=0.7，值越小越确定

// 重复惩罚：防止AI变成复读机
let penalty = 1.1f32;
for &id in &self.token_ids {
    let idx = id as usize;
    if idx < logits_vec.len() {
        if logits_vec[idx] > 0.0 {
            logits_vec[idx] /= penalty;  // 正logits除以惩罚系数
        } else {
            logits_vec[idx] *= penalty;  // 负logits乘以惩罚系数
        }
    }
}
```

**temperature=0.7**：平衡创意与稳定性，不至于太随机

**repetition_penalty=1.1**：轻微惩罚重复token，避免"啊啊啊啊"的死循环

### 2.4 WebGPU的遗憾与WASM SIMD的救赎

这个Demo用的是**CPU推理**。

目前 `candle` 框架对 WebGPU 后端（尤其是 BF16 精度）的支持尚未完全成熟，无法正常使用。

为了保证 100% 的兼容性，本 Demo 采用了 **CPU 推理 + WASM SIMD 加速**。在笔记本上，0.5B 模型已能达到非常丝滑的吐字速度。

WASM SIMD（128位向量化指令）给CPU推理显著加速。目前在高版本的浏览器中，WASM SIMD已是标准配置，默认开启。

可在浏览器的控制台中输入如下代码来验证：

```javascript
// 验证一个简单的 SIMD 模块：定义一个返回 v128 向量的函数
// 返回 true 就说明已经开启
WebAssembly.validate(
  new Uint8Array([
    0,
    97,
    115,
    109,
    1,
    0,
    0,
    0, // Wasm Header
    1,
    5,
    1,
    96,
    0,
    1,
    123, // Type: () -> v128 (0x7b)
    3,
    2,
    1,
    0, // Function: idx 0
    10,
    8,
    1,
    6,
    0,
    65,
    0,
    253,
    15,
    11, // Code: i32.const 0, i16x8.splat, end
  ]),
);
```

如没有，也可以在Chrome/Edge里， 通过`--enable-features=WebAssemblySimd`来开启。

**未来可期**：等 `candle` 框架完善 WebGPU 后端后，只需改一行代码`Device::Cpu`→`Device::WebGpu`，速度预计就能翻5-10倍。

## 三、3D形态：VRM格式与Three.js渲染

### 3.1 VRM给了GLB灵魂

如果说GLB是3D模型的身体，那**VRM就是二次元的灵魂协议**。

| 特性     | GLB（通用） | VRM（二次元专用）                      |
| -------- | ----------- | -------------------------------------- |
| 骨骼命名 | 随意        | 强制Humanoid标准（Hips/Spine/Head）    |
| 表情控制 | 无标准      | 统一BlendShape命名（Joy/Blink/Sorrow） |
| 材质     | PBR写实     | 支持MToon卡通渲染                      |
| 兼容性   | 所有软件    | VRoid/Blender二次元工具链              |

代码加载3D模型后，第一件事就是**按标准名字自动捕获骨骼**：

```javascript
// digital_human.js - 骨骼自动绑定
this.model.traverse((child) => {
  if (child.isBone) {
    const lowName = child.name.toLowerCase();
    if (lowName.includes("neck")) this.bones.neck = child;
    if (lowName.includes("head")) this.bones.head = child;
    if (lowName.includes("pelvis")) this.bones.hips = child;
    // 右手捂笑专用骨骼
    if (lowName.includes("upperarm_r")) this.bones.upperArmR = child;
    if (lowName.includes("hand_r")) this.bones.handR = child;
  }
});
```

正因为VRM的强制规范，无论模型来自VRoid还是Blender，只要符合标准，我们都能一键驱动。

### 3.2 Three.js实战：让角色活起来

**3D场景三件套**：

```javascript
// index.js - 初始化3D场景
const scene = new THREE.Scene();
// 程序化生成蓝天白云草地背景
scene.background = createProceduralBackground();

const camera = new THREE.PerspectiveCamera(45, w / h, 0.1, 1000);
camera.position.set(0, 1.0, 0.65); // 正面对视，营造陪伴感

const light = new THREE.DirectionalLight(0xffffff, 2.0);
light.position.set(2, 4, 5); // 斜上方主光源
```

**动画循环**：在`update()`里用正弦波模拟呼吸、眨眼、轻微晃动：

```javascript
// digital_human.js - 动画混合逻辑
update() {
    const time = Date.now() * 0.001;

    // 1. 全身呼吸起伏
    this.model.position.y = Math.sin(time * 1.0) * 0.01;

    // 2. 颈部自然扭动
    this.bones.neck.rotation.x = 0.1 + Math.sin(time * 0.5) * 0.12;

    // 3. 随机眨眼（用正弦波模拟概率）
    const blink = Math.sin(time * 4);
    this.setExpression("blink", blink > 0.98 ? 1.0 : 0);

    // 4. 捂嘴笑循环（核心卖点）
    const liftProgress = this.calculateLiftCycle(time); // 周期约14秒
    this.bones.upperArmR.rotation.z =
        1.3 * (1 - liftProgress) + 0.7 * liftProgress;
}
```

**捂嘴笑动作拆解**：

- **周期约14秒**：抬起→保持→放下→空闲，节奏自然不机械
- **手臂用球面插值**：大臂、前臂、手腕三段联动，轨迹自然
- **手指微握**：避免僵尸手，保持优雅
- **表情同步**：微笑(`Fcl_MTH_Joy`) + 眯眼(`Fcl_EYE_Close`)同时触发

**BlendShape控制**：

```javascript
// digital_human.js - 表情映射逻辑
const VROID_MAP = {
  mouthOpen: "Fcl_MTH_A", // 对应“啊”口型
  happy: "Fcl_ALL_Joy",
  blink: "Fcl_EYE_Close",
};

setExpression(key, value) {
  const targetName = VROID_MAP[key];
  this.morphMeshes.forEach((mesh) => {
    const index = mesh.morphTargetDictionary[targetName];
    if (index !== undefined) {
      mesh.morphTargetInfluences[index] = value;
    }
  });
}
```

## 四、跨界协同：让 AI 真正开口说话

### 4.1 Wasm与JS的丝滑通信

Rust每生成一个Token，通过Worker消息零拷贝传输：

```javascript
// worker.js - Token流式输出
while (!engine.is_finished()) {
  const token = engine.step(); // Rust函数，返回String
  self.postMessage({ action: "token", token });
  // 让出线程，防止卡死主线程
  await new Promise((r) => setTimeout(r, 0));
}
```

JS层收到后立即拼接并触发语音：

```javascript
// index.js - Token接收与TTS同步
case "token":
    currentFullResponse += token;
    tts.append(token); // 流式输入，不等说完就发音
    avatar.setSpeaking(true); // 立即开启嘴型动画
```

LLM 吐字很快，但语音合成需要整句才自然。我们在 `tts.js` 中设计了一个缓冲区，**按标点符号断句**，实现**边吐字边发声**:

```javascript
// tts.js - 按标点切分句子

// 包含逗号、顿号在内的所有停顿符号都直接触发播放
const punctuation = /[。！？，；.!?,;]/;
let match = punctuation.exec(this.buffer);

if (match) {
  const sentence = this.buffer.substring(0, match.index + 1).trim();
  this.buffer = this.buffer.substring(match.index + 1);
  if (sentence) this.enqueue(sentence);
  return;
}
```

### 4.2 Web Speech API：浏览器的原生声优

不需要服务器，浏览器自带TTS引擎：

```javascript
// tts.js - 自动选择中文女声
initVoice() {
    const voices = this.synth.getVoices();
    this.voice = voices.find(v =>
        v.lang.includes("zh") && v.name.includes("美嘉")
    ) || voices.find(v => v.lang.includes("zh")) || voices[0];

    utterance.rate = 1.0; // 语速
    utterance.pitch = 1.1; // 音调拉高一点，更像少女
}
```

**语音状态驱动3D动画**：

在 `index.js` 的渲染循环中，我们根据语音播放状态（`isSpeaking`）实时扰动口型。虽然简单，但在 0.5B 模型的低延迟配合下，视觉效果惊人地自然：

```javascript
function render() {
  requestAnimationFrame(render);
  const time = Date.now() * 0.001;

  if (avatar.isSpeaking) {
    // 利用正弦函数模拟嘴部的自然开合
    const mouthA = (Math.sin(time * 15) + 1) * 0.4;
    avatar.setExpression("mouthOpen", mouthA);
  } else {
    avatar.setExpression("mouthOpen", 0);
  }
  avatar.update(); // 内部处理随机眨眼与身体微晃
}
```

## 五、工程总结与展望

虽然目前 0.5B 模型的智商还不能带你写复杂的架构设计，但这套架构已经彻底打通了**大模型推理+高性能 3D+实时语音**的 Web 全栈路径。

**当前挑战：**

- **内存压力**：0.5B 模型仍需占用约 3GB 内存，移动端仍有压力。
- **口型精度**：目前的嘴动基于随机波动，尚未实现真正基于语音频率（FFT）的元音匹配。

**未来进化：**

- **WebGPU 推理**：跟进 `candle` 更新，实现基于WebGPU的推理，目标是 5-10 倍的推理提速。
- **情感驱动**：让 LLM 输出表情指令（如“开心”），直接驱动 3D 模型做出转圈、捂嘴等全身动作。

## 六、开源与复现：把“她”带回家

- 完整代码已开源: https://github.com/Doomking/rust-zhixingshe-examples/tree/main/wasm/candle-llm-3d
- LLM模型：
  下载model.safetensors、config.json、tokenizer.json等文件到 `www/assets/models`目录
  - Qwen2.5-0.5B-Instruct： https://hf-mirror.com/Qwen/Qwen2.5-0.5B-Instruct
  - Qwen1.5-0.5B-Chat：https://hf-mirror.com/Qwen/Qwen1.5-0.5B-Chat

- 3D模型：
  访问 https://hub.vroid.com/en ，搜索关键词，然后通过 `Filter by conditions of use`, 筛选 `Download: Allow` 的模型，即可以进行下载, 下载后保存到 `www/assets/human` 目录。

**上手步骤**：

```bash
# 1. 克隆项目
git clone https://github.com/Doomking/rust-zhixingshe-examples.git
cd wasm/candle-llm-3d

# 2. 下载LLM模型

# 3. 下载3D模型

# 4. 编译Rust到WASM
wasm-pack build --release --target web

# 5. 本地运行（需Python 3）
python -m http.server 8080
# 或者用 Rust 的 miniserve
miniserve . --index www/index.html

# 6. 打开浏览器访问
# http://localhost:8080/wwww/index.html
```

## 七、写在最后：技术浪漫主义

这个项目最大的价值，不只是证明了浏览器能跑LLM，而是**让AI有了温度**。

当你看到3D角色在向你说话、捂嘴轻笑时，你会突然意识到：Rust的严谨、Wasm的跨界、Three.js的灵动，这些冰冷的工具合在一起，竟然能创造出有情感共鸣的"生命体"。

这就是技术浪漫主义——用代码编织梦想，让二次元走进现实。
