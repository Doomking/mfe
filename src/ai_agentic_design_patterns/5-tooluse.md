# AI智能体设计模式 05：从对话式AI到执行式AI——工具调用让Agent真正动手干活

## 01 光会说话还不够

问 AI "今天北京的天气怎么样"，它洋洋洒洒说了一堆"通常情况下北京春季气温……"，就是不给你一个实际的温度。

这不是模型在糊弄你，是因为 LLM 本质上是个语言模型，只能基于训练数据生成文字，没办法真正去查实时信息。

**工具调用（Tool Use）** 就是解决这个问题的。

## 02 工具调用是怎么回事

简单说，工具调用让 LLM 能够调用外部函数来获取它自己不知道的信息，或者执行它自己做不到的操作。

不加工具的 LLM 是封闭的：只能凭训练数据回答，不会的就不会。加了工具调用之后，它可以主动去查天气、查数据库、发消息、跑代码——需要什么就调什么。

![pattern.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/543780bfd3fb4a1a89a7d9d96c9660b0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgUnVzdOefpeihjOekvg==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDE4NjU3MjAxOTczNzYyNCJ9&rk3s=e9ecf3d6&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1773891901&x-orig-sign=q33r7ryBHrpZmOYrEUFnuqtAspg%3D)
整个流程是这样的：

```
用户提问
   ↓
LLM 判断：需要外部信息吗？
   ↓ 需要
生成结构化调用请求（JSON）
   ↓
框架/运行时执行外部函数
   ↓
将结果返回给 LLM
   ↓
LLM 整合结果，生成最终回答
   ↓
用户拿到有用的答复
```

有一点要理解：**LLM 本身不执行代码**。它只是告诉框架"我要调用哪个函数、参数是什么"，真正去跑代码的是运行时框架。这个分工在 Rust Rig 的实现里体现得很清楚。

## 03 工具调用能做什么

按用途大致分这几类：

| 类型     | 典型场景                   |
| -------- | -------------------------- |
| 信息检索 | 查天气、搜新闻、调搜索引擎 |
| 数据交互 | 查库存、查订单、读数据库   |
| 计算分析 | 做数学运算、跑统计脚本     |
| 通信操作 | 发邮件、发消息、推通知     |
| 代码执行 | 运行代码片段、测试逻辑     |
| 设备控制 | IoT 设备指令下发           |

说白了就一件事：**把 LLM 的判断能力，和外部世界的执行能力接起来**。

## 04 Rust 怎么实现

Python 有 LangChain、CrewAI，Rust 这边最值得用的是 **`rig`**（`rig-core`）。类型安全、异步支持好，工具调用的接口也设计得比较清晰。

下面用一个查天气的例子来看怎么定义工具：

**第一步：定义工具**

```
#[derive(Deserialize, Serialize)]
struct WeatherArgs {
    location: String,
}

#[derive(Deserialize, Serialize)]
struct WeatherTool;

impl Tool for WeatherTool {
    const NAME: &'static str = "get_weather";
    type Error = ToolError;
    type Args = WeatherArgs;
    type Output = String;

    // 告诉 LLM 这个工具是干什么的、参数怎么传
    async fn definition(&self, _prompt: String) -> ToolDefinition {
        serde_json::from_value(serde_json::json!({
            "name": "get_weather",
            "description": "Get the current weather for a specific location using a real API.",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city or location to get the weather for, e.g., 'London' or 'Paris'."
                    }
                },
                "required": ["location"]
            }
        }))
        .expect("Failed to parse tool definition")
    }

    // 实际执行逻辑：先查坐标，再查天气
    async fn call(&self, args: Self::Args) -> Result<Self::Output, Self::Error> {
        println!("Tool Called: get_weather for '{}'", args.location);

        let geo_url = format!(
            "https://geocoding-api.open-meteo.com/v1/search?name={}&count=1&language=en&format=json",
            args.location
        );
        let geo_res = reqwest::get(&geo_url).await
            .map_err(|_| ToolError("Failed to fetch geo data".to_string()))?;
        let geo_json: serde_json::Value = geo_res.json().await
            .map_err(|_| ToolError("Failed to parse geo JSON".to_string()))?;

        if let Some(results) = geo_json.get("results").and_then(|r| r.as_array()) {
            if let Some(first) = results.first() {
                let lat = first.get("latitude").and_then(|v| v.as_f64()).unwrap_or(0.0);
                let lon = first.get("longitude").and_then(|v| v.as_f64()).unwrap_or(0.0);
                let name = first.get("name").and_then(|v| v.as_str()).unwrap_or(&args.location);

                let req_url = format!(
                    "https://api.open-meteo.com/v1/forecast?latitude={}&longitude={}&current_weather=true",
                    lat, lon
                );
                let weather_res = reqwest::get(&req_url).await
                    .map_err(|_| ToolError("Failed to fetch weather data".to_string()))?;
                let weather_json: serde_json::Value = weather_res.json().await
                    .map_err(|_| ToolError("Failed to parse weather JSON".to_string()))?;

                if let Some(current) = weather_json.get("current_weather") {
                    let temp = current.get("temperature").and_then(|v| v.as_f64()).unwrap_or(0.0);
                    let wind = current.get("windspeed").and_then(|v| v.as_f64()).unwrap_or(0.0);
                    return Ok(format!("Weather in {}: {}°C, wind speed {} km/h", name, temp, wind));
                }
            }
        }

        Ok(format!("Could not find weather for {}", args.location))
    }
}
```

**第二步：把工具挂到 Agent 上**

```
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let _ = dotenv();
    let client: ollama::Client = ollama::Client::new(Nothing)?;

    let agent = client
        .agent("qwen2.5:14b")
        .preamble("You are a helpful assistant capable of using multiple tools to answer questions. \
                   When a tool provides a result, use it to answer the user's question accurately. \
                   If a tool result says that information is unavailable, inform the user.")
        .tool(WeatherTool)
        .tool(CapitalTool)
        .tool(GeneralInfoTool)
        .temperature(0.0)
        .build();

    let queries = vec![
        "What is the capital of France?",
        "What's the weather like in London?",
        "Tell me something about dogs.",
    ];

    for query in queries {
        println!("\nQuery: '{}'", query);
        match agent.prompt(query).await {
            Ok(response) => println!("Response: {}", response.trim()),
            Err(e) => println!("Error: {:?}", e),
        }
    }

    Ok(())
}
```

Agent 会自动判断每个问题需不需要调工具，需要就调，不需要就直接回答，整个过程不用手动控制。

**本地运行：**

```
# 安装 Ollama
brew install ollama

# 拉取模型
ollama pull qwen2.5:14b

# 运行示例
cargo run --example chapter_05_tooluse_example
```

**运行结果：**

```
Query: 'What is the capital of France?'
Tool Called: get_capital for 'France'
TOOL RESULT: The capital of France is Paris.
Response: The capital of France is Paris.

Query: 'What's the weather like in London?'
Tool Called: get_weather for 'London'
TOOL RESULT: Weather in London: 10.2°C, wind speed 14 km/h
Response: The weather in London is currently 10.2°C with a wind speed of 14 km/h.

Query: 'Tell me something about dogs.'
Tool Called: get_general_info for 'dogs'
TOOL RESULT: No specific information found, but the topic seems interesting.
Response: I couldn't find specific information about dogs, but dogs are often referred to
as man's best friend due to their loyalty. They come in various breeds and are known
for their intelligence and trainability.
```

完整代码：[GitHub 示例仓库](https://github.com/Doomking/rust-zhixingshe-examples/blob/main/ai-agent/agentic-design-patterns/agentic-design-patterns-rs/examples/chapter_05_tooluse_example.rs)

## 05 几个容易踩的坑

**工具描述要写清楚**

LLM 靠 `description` 来判断要不要调这个工具、怎么调。描述含糊就会调错工具或者传错参数。把工具描述当成 API 文档来写，越具体越好。

**参数类型要明确**

用 JSON Schema 约束参数类型，能减少 LLM 生成错误参数的情况。Rust 的类型系统在这里是个优势：反序列化失败就直接报错，不存在类型隐式转换这种隐患。

**工具权限要收窄**

工具的执行代码是开发者写的，LLM 并不能直接控制它做什么，但如果工具封装得太宽泛——比如直接暴露一个"执行任意 shell 命令"的接口——风险就很大了。原则是：**工具只做它声明的那一件事，不多不少**。

**处理好失败情况**

网络超时、API 报错、参数不合法，这些都会发生。框架通常会把错误信息传回给 LLM，让它重试或换个策略。但你自己的工具实现要有清晰的错误类型，别让 LLM 收到一堆模糊的错误信息不知道怎么处理。

## 06 写在最后

没有工具调用的 Agent，只是一个会聊天的程序。加上工具调用，它才能真正接入实际的业务流程，查数据、发消息、跑脚本，把决策落地成行动。

对 Rust 开发者来说，这个方向值得关注。Rust 在系统级开发、网络服务、WebAssembly 等场景已经很成熟，而这些场景正好是工具执行端最需要的：高性能、内存安全、可靠的异步调度。

AI 做判断，Rust 做执行，这个组合在实际工程中有不少可以探索的空间。

## 延伸阅读

- [Agentic Design Patterns — Tool Use](https://adp.xindoo.xyz/chapters/Chapter%205_%20Tool%20Use/)
- [rig-core 文档](https://docs.rs/rig-core/latest/rig/)
- [OpenAI Function Calling 官方文档](https://platform.openai.com/docs/guides/function-calling)

如果这篇对你有帮助，欢迎点赞转发。有问题或者想聊聊 Rust + AI 的实践，评论区见。

本文使用 [markdown.com.cn](https://markdown.com.cn) 排版
