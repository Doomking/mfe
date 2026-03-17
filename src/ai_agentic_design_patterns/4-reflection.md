# AI智能体设计模式 04：反思模式（Reflection）给 Agent 装上自检系统，让它学会自我纠错

你有没有遇到过这种情况：

让 AI 写段代码，它给你交付了一份看起来没问题的代码，但跑起来一堆报错。你把错误信息粘贴给它，它立刻回复："抱歉，这里有个问题，修改如下……"，然后给出了正确答案。

这时候你心里大概只有一个问题：**既然你知道怎么改，为什么第一次不给我正确的？**

这不是你选错了模型，这是大多数LLM应用的共同问题：**一次推理，输出就定了，不管对不对。**

要解决这个问题，就需要引入今天的主角——**反思模式（Reflection）** 。

## 01 反思模式是什么？

用一句话来说：反思模式就是给 AI 的输出结果加一道质检。

![pattern.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/789e3d9dc47b4ba89ade81ea98427a33~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1773813354&x-orig-sign=A94fJc6lmNdinnvUeXXRi6r6j6k%3D)
它不再是生成完就结束，而是形成一个闭环：**生成 → 评审 → 修正**。

整个过程就三步：

1.  **生成**——AI先写出初稿
1.  **评审**——另一个AI（或者同一个AI换个角色）来挑毛病
1.  **修正**——根据反馈改，改完再审，直到过关

反思模式的核心不是怎么写得更好，而是怎么把写出来的东西改得更好。

就像写文章，第一稿从来不是最好的，好内容都是一遍一遍改出来的。

## 02 为什么不直接把要求写清楚？

你可能会想：我把需求写得更详细一点，不就能一次出对吗？

理论上是这样，实际上很难。

举个例子，你让 AI 写一份产品需求文档，要求包括功能描述、技术方案、风险评估、时间规划。就算你把每一条都写进 Prompt，AI 也可能在某个地方写得不够深入，或者漏掉你没有明确说但默认应该有的内容。

复杂任务本来就很难一次描述完整。而反思模式的价值就在这里：**不需要你反复提要求，AI 自己会检查哪里没做好，然后去补。**

## 03 反思循环是怎么运转的？

反思模式通常由两个角色配合完成：

**生产者 Agent**

- 负责生成内容，比如写代码、写文章、做方案
- 关注点是把事情做出来

**评审者 Agent**

- 负责挑问题，提改进意见
- 关注点是把问题找出来

工作流程如下：

```
用户提需求 → 生产者生成初稿 → 评审者审查 → 反馈给生产者 → 生产者修改 → 再审查 → ... → 输出终稿
```

这个过程不是无限循环的，通常会设置最大迭代次数（比如3轮），或者当评审者认为"没问题了"时自动结束。

为什么要把生产者和评审者分开？因为自己检查自己的东西，很容易看漏。同一段代码，换个角度来审，才更容易发现问题。

## 04 哪些场景最适合用反思模式？

反思模式在这几个方向效果很明显：

**代码生成与调试**：这是最适合的场景。写出代码后，通过运行测试或静态分析发现问题，再根据反馈自动修复，最终产出更可靠的代码。

**创意写作与文案**：先出草稿，再检查语气、流畅度和逻辑，反复打磨，让输出更接近真人的写作水准。

**复杂推理问题**：多步骤推理时，每一步都检查是否引入了新的矛盾，发现问题及时回头调整。

**长文档摘要**：生成摘要后，检查是否漏掉了关键信息或者存在事实偏差，确保最终结果准确且完整。

**计划与策略制定**：先制定行动计划，再模拟执行过程，提前发现潜在的问题点，避免执行时才踩坑。

**对话机器人**：客服机器人在回复之前，先检查这个回答是否和上下文一致，是否真的解决了用户的问题。

## 05 用 Rust + Rig 实现反思模式

下面用一个具体例子来说明怎么实现。

场景是：**让 AI 写一个 Python 函数，然后让另一个 AI 来审查这段代码有没有问题，如果有问题就反馈给前者修改，直到审查通过。**

**第一步：定义任务**

```
// 核心任务描述
let task_prompt = "Your task is to create a Python function named `calculate_factorial`. \
This function should do the following: \
1. Accept a single integer `n` as input. \
2. Calculate its factorial (n!). \
3. Include a clear docstring explaining what the function does. \
4. Handle edge cases: The factorial of 0 is 1. \
5. Handle invalid input: Raise a ValueError if the input is a negative number.";
```

**第二步：定义两个Agent**

```
// 生产者 Agent：负责写代码
let generator_agent = client
    .agent("qwen2.5:14b")
    .preamble("You are a helpful assistant that writes clean, efficient Python code.")
    .temperature(0.1)
    .build();

// 评审者 Agent：负责找问题
let reflector_agent = client
    .agent("qwen2.5:14b")
    .preamble(
        "You are a senior software engineer and an expert in Python. \
        Your role is to perform a meticulous code review. \
        Critically evaluate the provided Python code based on the original task requirements. \
        Look for bugs, style issues, missing edge cases, and areas for improvement. \
        If the code is perfect and meets all requirements, respond with the single phrase 'CODE_IS_PERFECT'. \
        Otherwise, provide a bulleted list of your critiques."
    )
    .temperature(0.1)
    .build();
```

**第三步：反思循环**

```
for i in 0..max_iterations {
    // 阶段1：生成/改进
    let generation_prompt = if i == 0 {
        task_prompt.to_string()
    } else {
        format!(
            "{}\n\nPlease refine the code using the critiques provided.",
            history_context
        )
    };

    let response = generator_agent.prompt(&generation_prompt).await?;
    current_code = response.clone();

    // 阶段2：评审
    let review_prompt = format!(
        "Original Task:\n{}\n\nCode to Review:\n{}",
        task_prompt, current_code
    );
    let critique = reflector_agent.prompt(&review_prompt).await?;

    // 阶段3：判断是否结束
    if critique.contains("CODE_IS_PERFECT") {
        println!("评审通过，无需继续修改。");
        break;
    }

    // 记录本轮历史，供下一轮参考
    history_context.push_str(&format!(
        "\nIteration {} Code:\n{}\nCritique:\n{}\n",
        i + 1, current_code, critique
    ));
}
```

**第四步：本地运行**

这个示例用 Ollama 在本地跑，不需要调用云端 API，数据不出本地，成本几乎为零。

```
# 安装 Ollama
brew install ollama

# 拉取模型
ollama pull qwen2.5:14b

# 运行示例
cargo run --example chapter_04_reflection_example
```

**几个设计上值得注意的地方：**

- **角色分开**：生产者和评审者用不同的 system prompt，分别聚焦在"生成"和"审查"，不让它们互相干扰
- **多轮迭代**：可以持续循环，直到评审者认为没问题为止
- **完全本地**：用 Ollama 运行，不依赖任何云服务

完整代码参考：[GitHub 示例仓库](https://github.com/Doomking/rust-zhixingshe-examples/)

## 06 反思模式的几个坑

反思模式好用，但用的时候有几点要注意：

**成本会成倍增加**

每多一轮反思，就多调几次模型。迭代3轮，费用大约是原来的3倍。

应对方法：简单任务不用反思，只给复杂任务用。或者先用小模型生成初稿，再用大模型做最终审查，在成本和质量之间找平衡。

**评审标准不清楚**

评审者如果没有明确的检查标准，可能挑一些无关紧要的问题，或者反而漏掉了真正的问题。

应对方法：给评审者一个具体的检查清单，比如"必须检查以下5点"，让它有据可依。

**陷入反复拉锯**

生产者改了，评审者又挑新问题，循环停不下来。

应对方法：设置最大迭代次数，到了上限就强制结束。也可以引入第三个仲裁角色，在双方僵持时做最终判断。

## 写在最后

反思模式的本质，不是让 AI 变得更聪明，而是让 AI 的工作方式更接近人处理事情的方式。

好文章是改出来的，好代码是 Review 出来的，好方案是讨论出来的。AI也一样。

如果你现在有个Agent一直输出不稳定，不妨给它加一轮反思试试，效果往往比换模型更直接。

有兴趣的话可以：

1.  克隆文中的 [GitHub 仓库](https://github.com/Doomking/rust-zhixingshe-examples/)，在本地跑通这个例子
1.  把反思模式接入你现有的Agent，看看输出质量有没有变化
1.  试试用小模型生成、大模型审查的分层方案，感受一下成本和效果的差异

欢迎在评论区聊聊你的实践经验。
