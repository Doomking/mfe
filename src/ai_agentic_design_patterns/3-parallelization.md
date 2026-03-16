# AI智能体设计模式 03：从串行到并行，让 Agent 执行效率翻倍

你的Agent执行流程是这样的吗？

用户问一个问题，LLM要依次调用工具A，再调用工具B，最后调用工具C。中间任何一步卡住，整个流程都得等着。几十秒过去，用户早就关掉页面了。

问题不在于模型，而在于执行方式。

提速的秘诀不是换更大的模型，而是用 **并行模式（Parallelization）** 重构你的Agent。

这是Agentic Design Patterns里最实用的性能优化手段，今天咱们一次讲清楚。

## 01 并行为什么这么厉害？

传统的Agent工作流，就是按顺序一个接一个地干活。任务再多，也得按步骤来，前面的做不完，后面的就需要等。

并行模式的核心在于把那些可以独立完成的任务拆开，让它们同时跑。

![parallel.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/6226437455904b88b64b07864014551b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1773721662&x-orig-sign=QJjsyOZgNiJUhD8I6aHyiNPJVDU%3D)

这么一想就很清楚了：

- **串行处理：** 耗时 = T1 + T2 + T3，哪一步卡住都得等
- **并行协同：** 耗时 = max(T1, T2, T3)，整体速度取决于最慢的那个任务

带来的不只是速度，还有稳定性的提升。一个分支出问题，其他分支照样能干活。

## 02 并行的两种玩法

并行不是简单开多线程，而是要讲究策略。

| 形态         | 用在什么场景   | 怎么玩                                   |
| ------------ | -------------- | ---------------------------------------- |
| **分片并行** | 任务能拆开     | 把大任务切成小任务，同时跑，最后合并结果 |
| **投票并行** | 需要多方面验证 | 调用多个模型/工具，综合不同意见做决策    |

简单说：分片并行是**分工合作**，投票并行是**集思广益**。

举个例子，代码审查的时候：

- 用分片并行：同时检查安全性、性能、代码风格三个维度
- 用投票并行：三个模型同时评审同一段代码，综合它们的结果

### 1. 分片并行：把任务拆开一起跑

把复杂任务拆成互不干扰的子问题，让它们同时执行。

- **怎么玩：** 拆任务 -> 并行跑 -> 合结果
- **什么时候用：** 写一篇长文章，5个Agent各写一个章节；处理长文档，切片后并行提取关键信息

### 2. 投票机制：多个角度验证

让多个模型（或多次调用）并行回答同一个问题，最后汇总。

- **怎么玩：** 多次推理 -> 对比结果 -> 综合判断
- **什么时候用：** 代码安全审计，3个Agent同时找Bug，取交集；高精度翻译，并行生成3个版本，选最好的

一句话总结：分片并行解决**快不快**的问题，投票并行解决**准不准**的问题。

## 03 这些场景最需要并行

实际用起来，这些地方并行效果特别好：

**1. 信息收集与研究**

比如做公司调研的Agent，并行搜索新闻、查股票数据、监控社交媒体、查数据库，比一个一个查询快得多。

**2. 数据处理与分析**

分析客户反馈时，可以同时做情感分析、提取关键词、分类反馈、识别紧急问题，一次性拿到多维度结果。

**3. 多API或工具调用**

旅行规划Agent可以同时查航班价格、酒店情况、当地活动、餐厅推荐，更快给出完整方案。

**4. 多组件内容生成**

写营销邮件时，同时生成主题、正文内容、找图片、写行动号召文本，最后组装成完整邮件。

**5. 验证与校对**

用户输入验证时，可以同时检查邮箱格式、手机号、地址是否匹配、内容是否合规。

**6. 多模态处理**

分析社交媒体帖子时，同时分析文本情感和图片内容，快速整合不同来源的信息。

**7. A/B测试或多选项生成**

生成创意文本时，可以用不同的提示同时生成多个标题，快速比较选出最好的。

## 04 用 Rust 实战一把

`为什么选 Rust？因为在高并发场景下，Rust 的异步运行时 Tokio 能让你告别Python 的全局解释器锁(GIL) 限制，多核 CPU 真正派上用场。`

### 第一步：配置 Gemini

先准备好 API 密钥：

```
# 在项目根目录创建.env文件
# 设置 Gemini API 密钥
GEMINI_API_KEY=你的API密钥
```

### 第二步：用Rust写并行代码

用`rig` Agent 框架配合 `rig::parallel!`，代码会简洁很多：

```
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 初始化 Gemini 客户端
    let gemini_client = gemini::Client::from_env();

    let topic = "The history of space exploration";
    let model_name = "gemini-3.1-flash-lite-preview";

    // 创建三个并行的Agent
    let summarize_agent = gemini_client
        .agent(model_name)
        .preamble("Summarize the following topic concisely:")
        .build();

    let questions_agent = gemini_client
        .agent(model_name)
        .preamble("Generate three interesting questions about the following topic:")
        .build();

    let terms_agent = gemini_client
        .agent(model_name)
        .preamble("Identify 5-10 key terms from the following topic, separated by commas:")
        .build();

    let synthesis_agent = gemini_client
        .agent(model_name)
        .preamble("Synthesize a comprehensive answer based on the provided information.")
        .build();

    // 构建并行处理链
    let chain = pipeline::new()
        .chain(parallel!(
            prompt(summarize_agent),
            prompt(questions_agent),
            prompt(terms_agent),
            passthrough()
        ))
        .map(|(summary, questions, terms, topic): (Result<String, _>, Result<String, _>, Result<String, _>, String)| {
            let sum_str = summary.unwrap();
            let q_str = questions.unwrap();
            let t_str = terms.unwrap();

            println!("--- [阶段 1] 并行任务完成 ---");
            println!(">> 摘要:\n{}\n", sum_str);
            println!(">> 问题:\n{}\n", q_str);
            println!(">> 关键词:\n{}\n", t_str);
            println!("--- [阶段 2] 开始综合生成最终回复 ---\n");

            format!(
                "Based on the following information:\n\
                 Summary: {}\n\
                 Related Questions: {}\n\
                 Key Terms: {}\n\
                 Synthesize a comprehensive answer for the Original topic: {}",
                sum_str, q_str, t_str, topic
            )
        })
        .chain(prompt(synthesis_agent));

    println!("\n=======================================================");
    println!("--- 开始并行执行 Rig 示例任务 ---");
    println!("主题: '{}'", topic);
    println!("=======================================================\n");

    let start = std::time::Instant::now();
    let response = chain.call(topic.to_string()).await?;
    let duration = start.elapsed();

    println!("\n=======================================================");
    println!("--- 并行最终回复 ---");
    println!("{}", response);
    println!("--- 并行执行耗时: {:.2?} ---", duration);
    println!("=======================================================\n");

    Ok(())
}
```

`Rust 的异步体系能充分压榨系统资源。如果需要并行10个甚至100个子任务，Rust 的内存安全和并发优势会让系统稳得很。`

**完整代码可以看这里：** [GitHub示例仓库](https://github.com/Doomking/rust-zhixingshe-examples/blob/main/ai-agent/agentic-design-patterns/agentic-design-patterns-rs/examples/chapter_03_parallelization_example.rs)

### 运行结果对比

在Mac M1上跑了一下，结果很能说明问题：

```
=======================================================
--- 并行执行结果 ---
--- [阶段 1] 并行任务完成 ---
>> 摘要:
Space exploration evolved from the mid-20th-century .....

>> 问题:
Here are three interesting questions about the history of space exploration:
1. The Geopolitical Catalyst: To what extent would the ......
2. The "Unsung" Pioneers: Beyond the famous names of astronauts and cosmonauts, ......
3. The Ethical Shift: How has the primary motivation for space exploration shifted since the 1960s, ......

>> 关键词:
Space Race, Apollo missions, artificial satellites, lunar landing, ......

--- [阶段 2] 开始综合生成最终回复 ---
=======================================================
--- 并行执行耗时: 7.97s ---
=======================================================
```

```
=======================================================
--- 串行执行结果 ---
=======================================================
--- 串行执行耗时: 14.35s ---
=======================================================
```

**性能对比：**

- **串行执行：** 14.35秒
- **并行执行：** 7.97秒
- **性能提升：** 约44%

并行处理在Mac M1环境下的优势很明显。

## 05 并行也有坑

并行模式效果是好，但用的时候得注意：

- **API限流风险：** 并行意味着瞬间请求量增加，记得加上速率限制，不然可能触发熔断
- **结果合并麻烦：** 拆开容易，合并难。建议用结构化输出（比如JSON）来规范返回格式，方便最后汇总
- **资源竞争：** CPU和内存不够时，并行可能反而变慢

## 06 写在最后

并行模式不只是技术优化，更是工程思维的转变。从确定性的顺序处理，到自适应的并发响应，这是AI工程化从Demo走向产品的关键一步。

想试试的话：

1.  **拉下代码跑一下：** 文中的GitHub仓库有完整实现，在本地跑通你的第一个Rust并行Agent
1.  **检查现有架构：** 看看你现在的Agent流程，有哪些地方可以并行？
1.  **交流讨论：** 你觉得Agent开发中，并行最难的地方是什么？是Token消耗太多，还是结果合并麻烦？欢迎在评论区聊聊！

## 参考资料

1.  Agentic Design Patterns - 并行模式详解：https://adp.xindoo.xyz/chapters/Chapter%203_%20Parallelization/
1.  Rust示例代码仓库（含完整实现）：https://github.com/Doomking/rust-zhixingshe-examples
1.  Rig框架文档：https://github.com/0xPlaygrounds/rig
1.  Gemini API文档：https://ai.google.dev/docs

本文使用 [markdown.com.cn](https://markdown.com.cn) 排版
