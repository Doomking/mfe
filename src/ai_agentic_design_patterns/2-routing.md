# AI智能体设计模式 02 - 别再手动调度了！用Routing设计模式，打造真正聪明的AI智能体

## 01 引言：当你的 Agent 遇到选择困难症

你是否曾遇到过这样的场景：AI 助手面对复杂任务时，总是执行固定流程，无法根据实际情况灵活调整？比如用户问"我想订一张去伦敦的机票"，AI却只会回答通用信息，而不是直接调用预订工具。这就是传统线性处理的痛点——缺乏自适应决策能力。

这正是单 Agent 的瓶颈。在复杂的工程实践中，试图用一个全能模型处理所有场景，不仅成本高、响应慢，更会导致逻辑坍塌。

**路由模式（Routing）** 就像是给 AI 加上了一个聪明的分拣中心，它能根据用户意图，将任务自动派发给最擅长的专家模型或工具。

今天，我们来聊聊智能体设计模式中的**路由模式（Routing）** ，它能让你的 AI 系统像人类一样，根据不同情况做出智能判断和选择。

## 02 什么是路由模式？

简单来说，它是一种让智能体系统根据输入、状态或前一操作结果，在多个潜在行动之间进行动态决策的机制。

路由模式的本质，是引入**条件逻辑**让 Agent 具备动态决策能力。它不再是死板地执行预设步骤，而是先`理解`请求，再`决定`走哪条路。

想象一下医院分诊台：患者进门，护士先判断是急诊、内科还是骨科，再引导到对应科室。路由模式就是你的 Agent 的分诊护士。


![routing_flow.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/184f2ba7f1db40148f52a0c61cd988b7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1773193798&x-orig-sign=gNTpkN3jUUenRuXqPafHh17N1W8%3D)

**关键转变：**

-   过去：固定 Pipeline，无论输入是什么都走同一套流程
-   现在：自适应响应，根据意图动态选择处理路径

这种模式的核心价值在于：

1.  **性能优化：** 简单问题走小模型（如 Qwen-1.5B），复杂问题走大模型。
1.  **专业分工：** 确保特定的子任务由装备了特定工具（tools）的 Agent 处理。
1.  **可靠性提升：** 避免无关上下文干扰 LLM 的判断。

## 03 拆解路由模式的四种心法

要实现高效路由，通常有四种主流机制：

1.  **基于 LLM 的语义路由：** 使用一个轻量级 LLM 作为分类器，判断意图并输出类别标签。这种方式灵活性最高，能处理复杂的语义。
1.  **基于嵌入（Embedding）的路由：** 将用户输入向量化，通过计算与预设类别向量的余弦相似度进行跳转。其优势在于极高的响应速度和极低的成本。
1.  **基于规则的路由：** 利用关键词匹配、正则表达或特定的 JSON 模式。虽然老土，但在处理格式化数据（如订单号、特定指令词）时最稳健。
1.  **基于传统机器学习模型的路由：** 当你有海量历史数据时，训练一个 SVM 或随机森林分类器，往往能平衡精度与推理耗时。

## 04 应用场景：它能帮你解决什么问题？

**1. 人机交互（智能客服/AI导师）** 用户说"我订单到哪了"→路由到订单查询、物流查询工具；"怎么退款"→路由到售后流程；"聊聊人生"→路由到闲聊模式。告别"答非所问"。

**2. 自动化数据处理管道** 邮件进来，路由判断是"销售线索"→进 CRM，"用户投诉"→升优先级，"垃圾邮件"→直接归档。RAG 系统也可用它做文档类型分发给不同解析器。

**3. 多智能体协作（Multi-Agent）** 研究型 Agent 中，路由根据任务类型分配给"搜索 Agent"、"总结 Agent"或"数据分析 Agent"。你是指挥官，Routing 是调度系统。

## 05 Rust 实战：用 Rig + Ollama 搭建路由网关

### 1. 本地环境准备

首先，确保你已经安装了 Ollama，并拉取了适合分类的轻量模型：

```
# 推荐使用通义千问 14B 版本，兼顾理解力与速度
ollama run qwen2.5:14b
```

### 2. 核心代码解析

我们的目标是构建一个**Pipeline Router**：

1.  **定义工具集**（BookingTool订机票、InfoTool查信息）
1.  **分类器Agent**（只输出`booker`/`info`/`unclear`）
1.  **Pipeline串联**（分类→重写prompt→调用对应工具）

```
// 完整逻辑见仓库完整示例
// https://github.com/Doomking/rust-zhixingshe-examples/blob/main/ai-agent/agentic-design-patterns/agentic-design-patterns-rs/examples/chapter_02_routing_example_cn.rs 

// 1. 定义工具：订机票
#[derive(Deserialize, Serialize)]
struct BookingTool;

impl Tool for BookingTool {
    const NAME: &'static str = "booker";
    type Error = ToolError;
    type Args = BookingArgs;
    type Output = String;

    async fn definition(&self, _prompt: String) -> ToolDefinition {
        serde_json::from_value(serde_json::json!({
            "name": "booker",
            "description": "预订机票或酒店",
            "parameters": {
                "type": "object",
                "properties": {
                    "request": {
                        "type": "string",
                        "description": "用户的具体预订请求。"
                    }
                },
                "required": ["request"]
            }
        }))
        .expect("Failed to parse tool definition")
    }

    async fn call(&self, args: Self::Args) -> Result<Self::Output, Self::Error> {
        println!("-------------------------- Booking Tool Called ----------------------------");
        Ok(format!("已为您模拟执行了以下预订操作：'{}'", args.request))
    }
}

// 2. 分类器Agent：只做意图识别，不处理业务
let classifier_agent = client
    .agent("qwen2.5:14b")
    .preamble(
        "你的任务是使用以下类别对用户的请求进行分类：[booker, info, unclear]\n\
            - booker: 用于预订机票、酒店或行程。\n\
            - info: 用于一般的知识性问题和信息检索。\n\
            - unclear: 如果请求不符合以上任何一类，或者意图不明确。\n\n\
            注意：只允许输出类别名称，不要输出任何其他多余字符。",
    )
    .temperature(0.0)
    .build();

// 3. 终极Agent：携带所有工具，等待被"激活"
let default_agent = client
    .agent("qwen2.5:14b")
    .preamble(
        "你是一个负责执行专用工具的智能体。 \
            关键规则：当调用工具获得了返回结果后，你必须完全按照字面意思一字不差地输出工具的原始结果。 \
            不要进行总结，不要添加任何开场白或结束语，也不要解释任何事情。只需返回最原始的工具输出。",
    )
    .tool(BookingTool)   // 装备订机票工具
    .tool(InfoTool)      // 装备查信息工具
    .temperature(0.0)
    .build();
```

**Pipeline的精髓** 在于 `map_ok` 这一步——根据分类结果动态重写 prompt，指挥终极 Agent 调用正确工具：

```
let chain = pipeline::new()
    // a. 提交给分类器进行初步意图判断
    .prompt(classifier_agent.clone())
    // b. 按照语义路由分配不同的执行 Prompt
    .map_ok(|x: String| match x.trim().to_lowercase().as_str() {
        "booker" => Ok(format!(
            "用户想要预订一些东西。你必须坚定地调用 'booker' 工具来处理此请求：'{}'",
            request
        )),
        "info" => Ok(format!(
            "用户正在询问信息。你必须坚定地调用 'info' 工具来处理此查询：'{}'",
            request
        )),
        "unclear" => Ok(format!(
            "用户的请求不清晰。请直接以助手的身份回复用户，并用中文要求他们澄清他们的请求：'{}'",
            request
        )),
        message => Err(format!("Could not process - received category: {message}")),
    })
    // c. 抹平错误嵌套层级 (解析前面的 Result)
    .map(|x| x.unwrap().unwrap())
    // d. 将携带不同重写规则的 Prompt 发送到包含各种工具的终极节点 Agent 进行触发
    .prompt(default_agent.clone());
match chain.try_call(*request).await {
    Ok(response) => println!("Final Output:\n {}", response.trim()),
    Err(e) => println!("Pipeline Failed: {:?}", e),
}
```

### 3. 运行及效果

#### 执行命令

```
ollama run qwen2.5:14b

cargo run --example chapter_02_routing_example_cn
```

#### 运行结果

```
--- Running Rig Pipeline Routing with Tools ---

--- Running with an booking request ---
Request: '帮我预订一张去伦敦的机票。'
-------------------------- Booking Tool Called ----------------------------
Final Output:
 已为您模拟执行了以下预订操作：'帮我预订一张去伦敦的机票。'

--- Running with an info request ---
Request: '意大利的首都是哪里？'
-------------------------- Info Tool Called ----------------------------
Final Output:
 正在处理信息请求：'意大利的首都是哪里？'。结果：已模拟检索该信息（请编造一个合理的回答）。意大利的首都是罗马。

--- Running with an unclear request ---
Request: '给我讲讲量子物理学。'
-------------------------- Info Tool Called ----------------------------
Final Output:
 正在处理信息请求：'给我讲讲量子物理学。'。结果：已模拟检索该信息（请编造一个合理的回答）。

--- Running with an random info request ---
Request: '告诉我一个随机的冷知识。'
-------------------------- Info Tool Called ----------------------------
Final Output:
 正在处理信息请求：'告诉我一个随机的冷知识。'。结果：已模拟检索该信息（请编造一个合理的回答）。

--- Running with an future booking ---
Request: '找一下下个月去东京的航班。'
-------------------------- Booking Tool Called ----------------------------
Final Output:
 已为您模拟执行了以下预订操作：'找一下下个月去东京的航班。'
```

## 06 挑战与应对方案

尽管路由模式强大，但在工程化中仍面临挑战：

| 挑战             | 缓解思路                           |
| -------------- | ------------------------------ |
| **分类错误级联**     | 增加不确定兜底分支，设置置信度阈值，低置信度转人工或澄清流程 |
| **延迟累积**       | 高频路径迁移到规则路由，LLM路由仅处理长尾意图       |
| **Prompt维护成本** | 分类prompt版本化管理，A/B测试不同分类策略      |
| **工具冲突**       | 工具命名语义化，prompt中明确工具使用场景，避免歧义   |

## 07 结语：迈向 Agentic 工作流

路由模式是 Agent 从玩具走向生产的分水岭。它让系统具备**上下文感知**和**动态决策**能力，是构建复杂AI应用的必经之路。

**接下来，你可以尝试：**

1.  **动手试用：** 克隆[示例仓库](https://github.com/Doomking/rust-zhixingshe-examples/blob/main/ai-agent/agentic-design-patterns/agentic-design-patterns-rs/examples/chapter_02_routing_example_cn.rs)，在本地用 Ollama 跑通上述代码。
1.  **优化路由：** 尝试将 `classifier_agent` 换成基于向量检索的静态路由。
1.  **加入讨论：** 在评论区分享你在设计路由模式时遇到的最大坑点。

### 参考资料

1.  [Agentic Design Patterns - Routing](https://adp.xindoo.xyz/chapters/Chapter%202_%20Routing/)
1.  [Rig 官方仓库](https://github.com/0xPlaygrounds/rig)
1.  [Ollama 文档](https://github.com/ollama/ollama)
1.  [Rust 知行社示例仓库](https://github.com/Doomking/rust-zhixingshe-examples)
