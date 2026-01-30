# Rust + Wasm + AI（二）：让浏览器开始思考 —— 基于 Candle 的端侧情感引擎

**导读**：上一篇我们聊了 Rust + Wasm + AI 的宏大愿景，这次我们真的把 BERT 模型塞进了浏览器。当用户在输入框敲下"这服务太棒了"的瞬间，模型已经完成推理，驱动满屏粒子爆发出青色光晕。全程零服务器请求，数据不出本地，甚至断网都能用。

## 1. 引言：打破 请求-响应 的旧枷锁

### 算力的南水北调

传统 AI 部署像南水北调 —— 把用户数据千里迢迢送到 GPU 集群，再把结果运回来。这种模式有三个问题：

1.  **延迟陷阱**：网络抖动100ms就能毁掉输入流畅感，复杂推理直奔秒级。
2.  **隐私裸奔**：每一句私密输入都在公网**裸奔**，数据必须出域。
3.  **成本黑洞**：简单分类任务也在消耗昂贵GPU显存，断网即服务死亡。

但算力格局正在发生剧变。M2 Max 的神经网络引擎已达 15.8 TOPS，高端安卓机的 NPU 也轻松突破 5 TOPS。

**Rust + Wasm 的出现，让我们能够实施算力突围：** 将推理任务下放到用户的 CPU/GPU。这不仅是成本的节约，更是人机交互体验的质变。

### 端侧 AI 的杀手锏

本次情感分析引擎在 MacBook Pro M1 的浏览器环境中可实现即时响应。

*   **零延迟**：用户松开键盘的瞬间，情感分数已出现在屏幕。
*   **隐私设计**：数据不出内存，连本地存储都不沾。
*   **离线优先**：一次加载，永久可用。

### 本篇核心

深度拆解如何利用 Rust 生态，让 **uer/roberta-base-finetuned-jd-binary-chinese** 模型在浏览器里实现毫秒级读心术。

## 2. 演示效果

先展示一下在浏览器中的运行效果：

![output1.jpg](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/34886ec669124d06a4dc683c31efeb72~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1769823376&x-orig-sign=JguplDHEJmHbNy%2BsCnTK6trgQa8%3D)

## 3. 技术选型：为什么是 Candle？

### Candle 的极致主义

**Candle** 是 HuggingFace 出品的纯 Rust 框架，专为**轻量化推理**而生， 关键优势在于：

1.  **真正的按需加载**：模型结构代码编译进 Wasm，权重按需 fetch，无冗余运行时。
2.  **零拷贝架构**：通过 `Safetensors` 格式，Rust 可以直接将 Wasm 内存映射为张量，无需在 JS 和 Rust 之间进行昂贵的序列化。
3.  **Wasm 友好**：纯 Rust 实现，无 C++ FFI，编译产物干净利落。

### Safetensors：Wasm 时代的权重协议

传统 PyTorch 的 `.bin` 格式，本质是 Pickle——**可以执行任意代码**，在浏览器里加载等于引狼入室。

**Safetensors** 是新标准，核心优势在于：

*   **零拷贝加载**：内存映射后直接算，无需反序列化。
*   **安全**：纯数据，无代码执行风险。
*   **自描述**：JSON 头信息让浏览器提前知道内存布局。

## 3. 架构设计：四层流水线

整个引擎分为四层，每层都是性能战场：

```bash
┌─────────────┐
│  资源层(fetch) │ ← 模型/分词器加载
├─────────────┤
│  转换层(Wasm)  │ ← 二进制流注入内存
├─────────────┤
│  计算层(Candle)│ ← 动态图构建与推理
├─────────────┤
│  交互层(Canvas)│ ← 粒子渲染与反馈
└─────────────┘
```

**关键设计决策**：

1.  **Tokenizer 预处理**：将 `tokenizer.json` 提前序列化为静态数组，避免运行时 JSON 解析开销。
2.  **零拷贝张量映射 (Zero-copy Mapping)**：利用 `Safetensors` 内存对齐 特性，将模型权重直接从 `ArrayBuffer` 映射为 `Candle` 张量，实现首屏启动零内存拷贝。
3.  **内存池复用**：推理中间结果复用同一块 Wasm 内存，避免 GC 压力。

***

## 4. 工程实战：从零构建 Wasm 推理核

### Python 端：模型选择与转换

最初选择的模型是 `jackietung/bert-base-chinese-finetuned-sentiment`，因为此模型支持  `.safetensors` 文件格式且支持中文环境。
但是试用过后，发现推理能力明显不行，比如输入 “难过”，结果推理结果为 “正向”。

`uer/roberta-base-finetuned-jd-binary-chinese` 模型, 对中文的情感分析能力更加强大，并且在京东真实评论数据上微调，电商场景准。

但是此模型好久没有更新过，没有提供 `.safetensors` 格式的文件，所以只能自己进行转换。

转换代码在 `candle-senti-pulse/model2safetensor` 目录中，是使用 `uv` 进行管理的Python脚本，用于将`uer/roberta-base-finetuned-jd-binary-chinese`模型转换为Safetensors格式。

代码实现如下：

```python
MODEL_NAME = "uer/roberta-base-finetuned-jd-binary-chinese"
SAVE_DIR = "./converted_model"

# 强制使用镜像
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"

def convert():
    if not os.path.exists(SAVE_DIR):
        os.makedirs(SAVE_DIR)

    try:
        # 加载
        model = AutoModelForSequenceClassification.from_pretrained(MODEL_NAME, trust_remote_code=True)
        tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME, trust_remote_code=True)

        # 1. 导出 config
        print("📦 正在生成 config.json...")
        config = model.config.to_dict()
        with open(os.path.join(SAVE_DIR, "config.json"), "w", encoding="utf-8") as f:
            json.dump(config, f, indent=2, ensure_ascii=False)

        # 2. 导出 tokenizer
        print("📝 正在生成 tokenizer.json...")
        tokenizer.save_pretrained(SAVE_DIR)

        # 3. 导出权重
        print("💾 正在生成 model.safetensors...")
        state_dict = model.state_dict()

        # 移除可能存在的 _orig_mod 等前缀（如果使用了 torch.compile）
        clean_state_dict = {k.replace("_orig_mod.", ""): v for k, v in state_dict.items()}

        save_file(clean_state_dict, os.path.join(SAVE_DIR, "model.safetensors"))

    except Exception as e:
        print(f"\n❌ 下载失败: {e}")

```

转换完成后，就可以将模型复制到 `www/models` 目录下。

### Rust 侧：SentiPulseEngine 设计

通过 Candle 加载模型的权重， 并使用 `VarBuilder` 构建计算图。

核心代码如下：

```rust
use candle_core::{DType, Device, Tensor};
use candle_nn::VarBuilder;
use candle_transformers::models::bert::{BertModel, Config};
use tokenizers::Tokenizer;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
#[derive(Debug)]
pub struct SentiPulseResult {
    negative: f32,
    positive: f32,
    neutral: f32,
}

#[wasm_bindgen]
impl SentiPulseResult {
    #[wasm_bindgen(getter)]
    pub fn negative(&self) -> f32 {
        self.negative
    }
    #[wasm_bindgen(getter)]
    pub fn positive(&self) -> f32 {
        self.positive
    }
    #[wasm_bindgen(getter)]
    pub fn neutral(&self) -> f32 {
        self.neutral
    }
}

#[wasm_bindgen]
pub struct SentiPulseEngine {
    model: BertModel,
    tokenizer: Tokenizer,
    // 分类头
    w_out: Tensor,
    b_out: Tensor,
    // 新增：Pooler 层 (用于处理 CLS 向量)
    w_pooler: Option<Tensor>,
    b_pooler: Option<Tensor>,
}

#[wasm_bindgen]
impl SentiPulseEngine {
    #[wasm_bindgen(constructor)]
    pub fn new(
        weights: &[u8],
        tokenizer_data: &[u8],
        config_str: &str,
    ) -> Result<SentiPulseEngine, JsError> {
        console_error_panic_hook::set_once();
        let device = &Device::Cpu;
        let tokenizer =
            Tokenizer::from_bytes(tokenizer_data).map_err(|e| JsError::new(&e.to_string()))?;
        let config: Config =
            serde_json::from_str(config_str).map_err(|e| JsError::new(&e.to_string()))?;
        let vb = VarBuilder::from_buffered_safetensors(weights.to_vec(), DType::F32, device)?;

        // 1. 加载 BERT
        let model = BertModel::load(vb.pp("bert"), &config)?;

        let w_pooler = vb
            .pp("bert")
            .get(
                (config.hidden_size, config.hidden_size),
                "pooler.dense.weight",
            )
            .ok();
        let b_pooler = vb
            .pp("bert")
            .get(config.hidden_size, "pooler.dense.bias")
            .ok();

        // 3. 加载 Classifier (带兼容逻辑)
        let num_labels = 2;

        // 使用 or_else 链式尝试不同的 Key 名
        let w_out = vb
            .get((num_labels, config.hidden_size), "classifier.weight")
            .or_else(|_| {
                vb.get(
                    (num_labels, config.hidden_size),
                    "classifier.out_proj.weight",
                )
            })
            .or_else(|_| vb.get((num_labels, config.hidden_size), "classifier.dense.weight"))
            .map_err(|_| JsError::new("权重文件中缺少分类层 (classifier weight)"))?;

        let b_out = vb
            .get(num_labels, "classifier.bias")
            .or_else(|_| vb.get(num_labels, "classifier.out_proj.bias"))
            .or_else(|_| vb.get(num_labels, "classifier.dense.bias"))
            .map_err(|_| JsError::new("权重文件中缺少分类层偏置 (classifier bias)"))?;

        Ok(Self {
            model,
            tokenizer,
            w_out,
            b_out,
            w_pooler,
            b_pooler,
        })
    }

    pub fn predict(&self, text: &str) -> Result<SentiPulseResult, JsError> {
        let device = &Device::Cpu;
        let tokens = self
            .tokenizer
            .encode(text, true)
            .map_err(|e| JsError::new(&e.to_string()))?;

        let input_ids = Tensor::new(tokens.get_ids(), device)?.unsqueeze(0)?;
        let token_type_ids = Tensor::new(tokens.get_type_ids(), device)?.unsqueeze(0)?;

        let enc = self.model.forward(&input_ids, &token_type_ids, None)?;

        // 取出 [CLS] (原始向量)
        let mut cls_token = enc.get(0)?.get(0)?.unsqueeze(0)?;

        // --- 执行 Pooler (如果存在) ---
        if let (Some(w), Some(b)) = (&self.w_pooler, &self.b_pooler) {
            // Pooler 逻辑: Tanh( Linear(CLS) )
            cls_token = cls_token.matmul(&w.t()?)?.broadcast_add(b)?.tanh()?;
        }

        // 执行分类计算
        let logits = cls_token
            .matmul(&self.w_out.t()?)?
            .broadcast_add(&self.b_out)?;

        let scale_factor = 1.0;
        let scaled_logits = (logits * scale_factor as f64)?;

        // 进行 Softmax
        let pr = candle_nn::ops::softmax(&scaled_logits.flatten_all()?, 0)?;
        let scores = pr.to_vec1::<f32>()?;

        // 自动适配二分类或三分类
        let (neg, pos, neu) = if scores.len() >= 3 {
            (scores[0], scores[1], scores[2])
        } else {
            let mut n = scores[0];
            let mut p = scores[1];
            let diff = (n - p).abs();

            // 基础中性分：当差距很大时，直接设为 0
            let mut m = if diff < 0.2 {
                0.8
            } else if diff < 0.4 {
                0.3
            } else {
                0.0 // 情绪明确，中性归零
            };

            // 执行归一化
            let total = n + p + m;
            if total > 0.0 {
                n = n / total;
                p = p / total;
                m = m / total;
                (n, p, m)
            } else {
                (0.33, 0.33, 0.34) // 兜底：防止除以零
            }
        };
        // 它们可能长这样：[0.1, 0.2, 0.5] -> 放大后 -> [0.5, 1.0, 2.5]
        web_sys::console::log_1(&format!("Raw Text: {}, Raw Scores: {:?}", text, scores).into());
        let result = SentiPulseResult {
            negative: neg,
            positive: pos,
            neutral: neu,
        };
        web_sys::console::log_1(&format!("Raw Text: {}, result: {:?}", text, result).into());

        Ok(result)
    }
}

```

**关键行解析：**

*   **`VarBuilder::from_buffered_safetensors`**: 避免了将巨大的模型文件在内存中反复 Copy，直接在内存池中构建权重。
*   **`Result<T, JsValue>`**: 这是 Rust 与 JS 交互的最佳实践，能够让 JS 端的 `try-catch` 捕获到详细的 Rust 错误。
*   **`unsqueeze(0)`**: 将一维 `Token` 序列升维为模型所需的 `Batch` 张量。

### JS 侧：模型推理与粒子风暴

**粒子风暴系统**：
用于根据情感分析分数, 动态渲染不同的粒子颜色和速度。

```javascript
// =========================================
// PART 1: 粒子风暴系统 (Particle System)
// =========================================
const canvas = document.getElementById("particle-canvas");
const ctx = canvas.getContext("2d");

// 设置画布大小
function resizeCanvas() {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
}
window.addEventListener("resize", resizeCanvas);
resizeCanvas();

// 粒子参数全局状态 (受 AI 情绪驱动)
let globalMood = {
  neg: 0.1, // 初始平静状态
  pos: 0.9,
  neu: 0.1,
  targetSpeed: 0.5,
  currentSpeed: 0.5,
  chaos: 0.2, // 混乱度
};

class Particle {
  constructor() {
    this.reset();
    this.y = Math.random() * canvas.height; // 初始随机分布
  }

  reset() {
    this.x = Math.random() * canvas.width;
    this.y = canvas.height + Math.random() * 100; // 从底部生成
    this.size = Math.random() * 2 + 1;
    // 基础速度 + 随机扰动
    this.baseSpeedY = Math.random() * 1 + 0.5;
    this.vx = (Math.random() - 0.5) * 0.5;
    this.vy = -this.baseSpeedY;
    this.alpha = Math.random() * 0.5 + 0.2;
  }

  update() {
    // 根据全局情绪平滑调整当前速度
    globalMood.currentSpeed +=
      (globalMood.targetSpeed - globalMood.currentSpeed) * 0.05;

    // 情绪越消极，速度越快，水平扰动越大(混乱)
    this.x += this.vx * (1 + globalMood.chaos * 5);
    this.y += this.vy * globalMood.currentSpeed;

    // 边界检查，循环利用
    if (this.y < -10) this.reset();
  }

  draw() {
    /// 强化颜色计算：确保 neg 占主导时 R 通道强制拉满
    const r = Math.floor(globalMood.neg * 255 + globalMood.neu * 168);
    const g = Math.floor(globalMood.pos * 242 + globalMood.neu * 85);
    const b = Math.floor(globalMood.pos * 255 + globalMood.neu * 247);

    // 氛围补偿：负面越高，粒子稍微变大一点，增加压迫感
    const dynamicSize = this.size * (1 + globalMood.neg * 1.5);

    ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${this.alpha + globalMood.neg * 0.3})`;
    ctx.beginPath();
    ctx.arc(this.x, this.y, dynamicSize, 0, Math.PI * 2);
    ctx.fill();
  }
}

const particles = Array.from({ length: 150 }, () => new Particle());

function animateParticles() {
  // 使用半透明黑色清空画布，制造拖尾效果
  ctx.fillStyle = "rgba(10, 11, 16, 0.2)";
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  particles.forEach((p) => {
    p.update();
    p.draw();
  });
  requestAnimationFrame(animateParticles);
}

// 启动粒子动画
animateParticles();
```

**模型初始化**：

通过 `fetch` 加载模型资源，并实例化。

```javascript
// 中文模型资源
const baseUrl = "./model/uer/roberta-base-finetuned-jd-binary-chinese/";

// 注意：文件名可能需要根据仓库实际情况调整
const [weights, tokenizer, config] = await Promise.all([
  fetch(baseUrl + "model.safetensors").then((r) => r.arrayBuffer()),
  fetch(baseUrl + "tokenizer.json").then((r) => r.arrayBuffer()),
  fetch(baseUrl + "config.json").then((r) => r.text()),
]);


const engine = new SentiPulseEngine(
  new Uint8Array(weights),
  new Uint8Array(tokenizer),
  config,
);
```

#### 模型推理

通过输入中文评论，调用 Rust Wasm 模块进行情感分析, 并返回情感分数。

```javascript
const t0 = performance.now();
// 1. 调用 Rust Wasm (返回对象包含 neg, pos, neu)
const result = engine.predict(text);
const { negative: neg, positive: pos, neutral: neu } = result;
const t1 = performance.now();
```

## 5. 运行情感分析推理

```bash
# 1. 模型下载及转换
cd model2safetensor && uv run main.py
# 2. 构建 Wasm 模块
cargo build --target web --release
# 3. 启动本地服务器
miniserve .
```

访问 <http://127.0.0.1:8080/www/index.html> ，在输入框输入中文评论，就可以看到情感分析分数及情绪粒子风暴的变化了。

## 6. 总结：开启 Web 推理的新纪元

从 调包侠 到 推理架构师, 这一步的跨越在于：我们已可以**掌控算力的分配权。** 通过 Rust + Wasm，我们证明了即使是复杂的 Transformer 模型，也能在用户的指尖轻盈跃动。

下一步，我们将引入 **WebGPU**，探索如何在浏览器里运行 3B 参数量级的端侧大模型（LLM）。

**源代码地址**：<https://github.com/Doomking/rust-zhixingshe-examples/tree/main/wasm/candle-senti-pulse>
