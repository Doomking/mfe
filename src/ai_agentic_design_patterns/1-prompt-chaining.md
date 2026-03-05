# 智能体设计模式 01 - 提示链：别再盲目调优 Prompt！用 Rust + Rig 玩转提示链全实战

## 01 导语：为什么你的Prompt总在翻车？

你是否经历过这样的挫败，为了让大模型完成一个复杂任务（如：分析财报 → 提取数据 → 撰写邮件），你写了一段长达两页纸的终极 Prompt，结果模型要么漏掉了关键信息，要么输出格式混乱。

**这不是模型不够强，而是认知负荷过载**

在大模型（LLM）开发领域，这种试图大力出奇迹的做法正逐渐失效。**单次调用（Single Prompt）的认知负荷上限，正是AI幻觉与不稳定的温床。**

吴恩达（Andrew Ng）早就指出：与其追求一个完美的Prompt，不如构建一个能够迭代、拆解的智能体流水线（Agentic Workflow）。今天，我们聊聊智能体设计模式的开篇之作——**提示链（Prompt Chaining）**。

## 02 什么是提示链？

**提示链（Prompt Chaining）**，有时也被称为**管道模式（Pipeline Pattern）**，简单来说就是「把复杂任务拆成多个小步骤，一步一步解决」的AI工作模式。

核心思想是**分而治之**，它不期望模型在一个步骤中解决所有问题，而是将复杂任务拆解为一系列更小、更易管理的子任务。每个步骤的输出作为下一步的输入，通过依赖关系建立起逻辑链条。

想象一下，你让一个人一次性完成「分析一份市场报告、提取关键数据、生成可视化图表、撰写总结邮件」，他很可能手忙脚乱。但如果把任务拆成四步，每一步只专注做一件事，效果就会好很多。

提示链就是这个道理，它不期望LLM一步到位解决复杂问题，而是采用「分而治之」策略：

- 将大任务拆解为一系列小任务
- 为每个小任务设计专门的提示
- 前一步的输出作为下一步的输入

这样做的好处是，每一步都更简单明确，模型认知负荷降低，输出质量自然就提高了！

## 03 结构解析：智能体流水线

一个完整的提示链就像一条生产线，每个环节都有明确的分工：

- **任务分解器**：像工厂的规划师，把复杂任务拆成一个个小任务
- **记忆/状态管理**：像传送带，在步骤间传递信息，确保上下文不丢失
- **决策器**：像质检员，根据前一步的结果决定下一步怎么做
- **执行器**：像工人，执行具体的LLM调用和数据处理
- **输出验证**：像品控，确保每一步的输出都符合质量标准

这样的结构化设计，让AI从「单打独斗」变成了「团队协作」，能力自然大幅提升。

## 04 具体应用场景：提示链的降维打击

提示链在实际应用中非常广泛，以下是7个典型场景：

### 1. 信息处理工作流

**适用场景**：分析大量文档、生成研究报告、自动化内容分析、AI驱动的研究助手开发
**具体例子**：

- 第一步：从给定的URL或文档中提取文本内容
- 第二步：总结清洗后的文本
- 第三步：从总结或原文中提取特定实体（如姓名、日期、地点）
- 第四步：使用这些实体搜索内部知识库
- 第五步：结合总结、实体和搜索结果，生成最终报告

### 2. 复杂问答系统

**适用场景**：需要多步推理的专业问题、多步信息检索
**具体例子**：

- 第一步：识别用户查询中的核心子问题
- 第二步：分别收集每个子问题的信息
- 第三步：整合信息生成最终答案

### 3. 数据提取与转换

**适用场景**：处理发票、表单等非结构化数据、OCR处理
**具体例子**：

- 第一步：从发票中提取特定字段（如姓名、地址、金额）
- 第二步：验证字段完整性和格式
- 第三步：如果字段缺失或格式不正确，专门查找缺失信息
- 第四步：再次验证结果，如有必要重复
- 第五步：提供经过验证的结构化数据

### 4. 智能内容生成

**适用场景**：撰写文章、报告、营销文案、创意故事、技术文档
**具体例子**：

- 第一步：基于用户的兴趣爱好，生成多个主题
- 第二步：允许用户选择一个主题或自动选择最好的一个
- 第三步：基于选定的主题，生成详细的大纲
- 第四步：基于大纲的各点撰写草稿，提供前一部分作为上下文
- 第五步：审阅和润色完整的初稿，确保连贯性、语气和语法

### 5. 有状态的对话智能体

**适用场景**：多轮对话系统、需要保持上下文的交互
**具体例子**：

- 第一步：处理用户的第一轮发言，识别意图和关键实体
- 第二步：更新对话状态，包含意图和实体
- 第三步：基于当前状态，生成响应或识别下一个所需的信息
- 第四步：在后续对话中重复此过程，利用不断积累的对话历史

### 6. 代码生成和优化

**适用场景**：AI辅助软件开发、代码编写和优化
**具体例子**：

- 第一步：理解用户的需求，生成伪代码或大纲
- 第二步：基于大纲，编写初始版本的代码
- 第三步：识别代码中可能存在的错误或需要改进的地方
- 第四步：基于识别出的问题，重写或优化代码
- 第五步：添加文档或测试用例

### 7. 多模态和多步推理

**适用场景**：分析包含多种模态（如图像、文本、表格）的数据
**具体例子**：

- 第一步：从用户的图像请求中提取并理解文本内容
- 第二步：将提取出的图像文本与其对应的标签进行关联
- 第三步：利用表格来解读已收集到的信息，以确定最终需要输出的内容

## 05 实战示例：Rust + Rig + Ollama 本地跑通

### 目标

用 Rust 实现一个两步骤的提示链，从非结构化文本中提取技术规格并转换为结构化JSON格式。

### 实现原理

我们将创建两个智能体：

1. **提取智能体**：从文本中提取技术规格
2. **转换智能体**：将提取的内容转换为结构化JSON

### 关键代码

```rust
// 初始化 Ollama 客户端（本地运行，无需API Key）
let client: ollama::Client = ollama::Client::new(Nothing)?;

// Step 1: 提取 Agent - 专门负责提取技术规格
let extractor_agent = client
    .agent("qwen2.5:14b")  // 使用 qwen2.5 模型
    .preamble(
        "You are a technical analyst. \
         Extract only the technical specifications from the user's input.",
    )
    .temperature(0.0)  // 设为0，确保输出稳定
    .build();

// Step 2: 转换 Agent - 专门负责格式转换
let transformer = client
    .extractor::<Specifications>("qwen2.5:14b")
    .preamble(
        "You are a JSON formatting assistant. \
         Convert the provided specifications into the required JSON structure.",
    )
    .build();

// 运行 Prompt Chain
let extracted_text = extractor_agent.prompt(input_text).await?;
let specs = transformer.extract(&extracted_text).await?;
```

### 运行步骤

1. **环境准备**：
   - 安装 Ollama：[https://ollama.com/download](https://ollama.com/download)
   - 下载模型：`ollama pull qwen2.5:14b`

2. **项目配置**：
   在 `Cargo.toml` 中添加依赖：

   ```toml
   [dependencies]
   rig = "0.1.0"
   dotenvy = "0.15.0"
   serde = { version = "1.0", features = ["derive"] }
   schemars = "0.8.16"
   tokio = { version = "1.0", features = ["full"] }
   ```

3. **运行命令**：

   ```bash
   # 启动 Ollama 服务
   ollama run qwen2.5:14b

   # 运行示例
   cargo run --example chapter_01_prompt_chaining_example
   ```

### 运行效果

输入：

```bash
The new laptop model features a 3.5 GHz octa-core processor, 16GB of RAM, and a 1TB NVMe SSD.
```

输出：

```json
{
  "cpu": "3.5 GHz octa-core",
  "memory": "16GB",
  "storage": "1TB NVMe SSD"
}
```

### 实战提示

Rig 的 `extractor` 配合 `schemars` 自动生成 JSON Schema，让 LLM 输出直接映射到 Rust 结构体，彻底告别手动解析。

## 06 选型对比：Rig vs LangChain-Rust

在 Rust 生态中，选型决定了你的开发体验：

| 维度         | **Rig** (推荐)                   | **LangChain-Rust**              |
| ------------ | -------------------------------- | ------------------------------- |
| **API 设计** | 简洁直观，Builder 模式，类型安全 | 宏驱动，深度模仿 Python 版      |
| **性能损耗** | 极低（零成本抽象）               | 较高（运行时动态分发）          |
| **上手难度** | 极低，Rust 开发者友好            | 中等，需理解 LangChain 核心概念 |
| **典型 API** | `extractor::<T>` 配合 `schemars` | 复杂的 `Chain` 与解析器组合     |
| **适用场景** | 高性能、定制化智能体             | 复刻 Python 工作流              |
| **扩展能力** | 模块化设计，易于扩展             | 插件系统完善，扩展性强          |

### 如何选择？

- **选 Rig**：如果你是 Rust 开发者，追求简洁高效，快速实现原型
- **选 LangChain-Rust**：如果你需要与其他语言版本的 LangChain 兼容，或需要更复杂的功能

### API 差异示例

```rust
// 1. 客户端初始化
// Rig - 简洁的 Nothing 类型
let client = ollama::Client::new(Nothing)?;
// LangChain-Rust
let ollama = Ollama::default().with_model("llama3.2");

// 2. Agent 构建
// Rig - Builder 模式，链式调用
let agent = client.agent("llama3.2").preamble("...").build();
// LangChain-Rust
let agent = ConversationalAgentBuilder::new()
        .tools(&[Arc::new(command_executor)])
        .options(ChainCallOptions::new().with_max_tokens(1000))
        .build(ollama)
        .unwrap();

// 3. 结构化输出
// Rig - 自动派生 JsonSchema，类型安全
let result: MyStruct = extractor.extract(input).await?;
// LangChain-Rust - 手动指定解析器
let executor = AgentExecutor::from_agent(agent).with_memory(memory.into());
let result = executor.invoke(input).await?.parse::<MyStruct>()?;
```

## 07 为什么你应该用 Rust 写 AI Agent？

- **极致性能**：零成本抽象让推理延迟降到最低，适合实时响应场景。
- **强类型安全**：借助 `schemars` 自动生成 JSON Schema，引导模型输出精准数据，编译通过即意味着运行时少踩坑。
- **并发优势**：`tokio` 异步生态能以极低开销处理高并发的提示链 I/O 请求。
- **本地私有化**：Rust 编译产物极小，与 Ollama 协同部署在本地，无需 API Key，更安全更快速。

## 08 优点与挑战

### 优点

- **可靠性**：分步处理降低了模型的认知负荷，输出质量显著提高
- **可维护性**：模块化设计让系统更易于理解和调试
- **灵活性**：可以根据需要调整提示链的长度和复杂度
- **可扩展性**：易于集成外部工具和数据源，功能无限扩展

### 挑战与解决方案

| 挑战                                   | 解决方案               |
| -------------------------------------- | ---------------------- |
| **延迟增加**：多步骤调用会增加响应时间 | 采用并行处理和缓存策略 |
| **错误传播**：一步出错影响整个链条     | 实现错误处理和重试机制 |
| **上下文丢失**：长链条可能丢失信息     | 设计合理的状态管理     |
| **成本增加**：多步骤调用增加 API 成本  | 优化提示和模型选择     |

在设计提示链时，合理控制链的长度，通常 3-5 步是最佳选择，过长会增加复杂性和延迟。

## 立即行动

**想获取更多 Rust + AI Agent 实战案例？**
欢迎关注 **「Rust 知行社」**，点击下方卡片，带你用 Rust 重新定义智能体。

1. **[尝试示例]**：克隆 [示例仓库](https://github.com/Doomking/rust-zhixingshe-examples/blob/main/ai-agent/agentic-design-patterns/agentic-design-patterns-rs/examples/chapter_01_prompt_chaining_example.rs) 并在本地跑通代码。
2. **[关注动态]**：Star Rig 和相关仓库，跟踪 Rust AI 生态进展。
3. **[加入讨论]**：在评论区分享：你在构建复杂 Prompt 时遇到的最大的问题是什么？

## 参考资料

1. [Rig 官方仓库](https://github.com/0xPlaygrounds/rig)
2. [Ollama 文档](https://github.com/ollama/ollama)
3. [LangChain-Rust](https://github.com/Abraxas-365/langchain-rust)
4. [Rust 知行社示例仓库](https://github.com/Doomking/rust-zhixingshe-examples)
5. [Agentic Design Patterns - Prompt Chaining 指南](https://github.com/ginobefun/agentic-design-patterns-cn/blob/main/07-Chapter-01-Prompt-Chaining.md)
