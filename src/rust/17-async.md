![](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/229bbc47caf34f2290418447f30ad5f8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Y-X5LmL5Lul6JKZ:q75.awebp?rk3s=f64ab15b&x-expires=1726280234&x-signature=U7x5IBo0%2BwITHIKd4PqOw80dXHM%3D)

# Rust异步编程：从入门到精通

在传统的同步编程中，当程序执行到一个可能会阻塞的操作时，比如网络请求或文件读取，线程会被阻塞，直到这个操作完成，这期间其他任务无法得到执行，极大地浪费了 CPU 资源，降低了程序的整体效率。

而异步编程则打破了这种限制，它允许程序在等待某个操作完成的同时，继续执行其他任务，从而提高了程序的响应速度和吞吐量。
在 Rust 中，异步编程主要通过Future和async/await这两个核心概念来实现，它们为开发者提供了一种简洁而高效的方式来编写异步代码。
## Future 与 async/await：异步编程的基石
### Future：未来的结果
在 Rust 的异步编程世界里，Future是一个非常重要的概念，它代表了一个异步操作的结果，只是这个结果在当下还处于未完成状态，就好像在网上预订了一件商品，下单之后，商品并不会立刻送到你手中，而是在未来的某个时间点才会送达。在商品送达之前，你可以去做其他事情，比如看电影、读书等。

Future也是如此，在它完成之前，程序可以继续执行其他任务，而不需要一直等待它的结果。当你需要获取这个异步操作的最终结果时，就需要主动去轮询（poll）这个Future ，直到它返回最终的结果。
Future的定义如下：
```rust
enum Poll<T> {
    Ready(T),
    Pending,
}
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```
Future是一个泛型 trait ，它的类型参数Output代表了异步操作的最终结果类型。

poll方法是Future的核心方法，它用于轮询Future的状态，返回一个Poll枚举类型的值。Poll枚举类型有两个可能的值：

- Pending表示异步操作还未完成;
- Ready则表示异步操作已经完成，返回最终的结果。

Future的实现需要满足以下要求：
1. 类型参数Output必须实现Send trait ，这意味着它可以在不同的线程之间安全地传递。
2. poll方法必须是可重入的，即多次调用poll方法不会改变Future的状态。
3. poll方法必须是非阻塞的，即它不会阻塞当前线程，而是立即返回一个Poll枚举类型的值。
### async/await：让异步代码更易读
async和await是 Rust 异步编程中的另外两个关键概念，它们的出现大大简化了异步代码的编写和阅读。

async用于定义一个异步函数，这个函数返回的是一个实现了Future trait 的类型。
例如，可以定义一个异步函数来模拟网络请求：
```rust
async fn fetch_data() -> String {
    // 模拟网络请求的耗时操作
    // 这里使用tokio::time::sleep来模拟等待
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
    "Hello, Rust!".to_string()
}
```

在这个例子中，fetch_data函数被async修饰，成为了一个异步函数。它内部使用了tokio::time::sleep来模拟网络请求的耗时操作，await关键字用于等待这个耗时操作完成。

当await一个Future时，当前的异步函数会暂停执行，直到被等待的Future完成，然后await会返回Future的结果。

#### 代码示例与实践

下面通过一个简单的代码示例来更深入地理解async/await的使用：

```rust
use tokio;

async fn print_message() {
    println!("开始异步任务");
    tokio::time::sleep(std::time::Duration::from_secs(2)).await;
    println!("异步任务完成");
}

#[tokio::main]
async fn main() {
    print_message().await;
}

```
在这个示例中，首先定义了一个异步函数print_message，在函数内部，先打印了一条开始异步任务的消息，然后使用tokio::time::sleep模拟了一个耗时 2 秒的异步操作，这里通过await等待这个操作完成，最后打印异步任务完成的消息。

在main函数中，同样是异步函数，调用了print_message并使用await等待其执行完成。这样，整个异步操作的流程就清晰地展现出来了，通过async/await，我们可以很方便地控制异步操作的执行顺序，让异步代码更加简洁明了。

## 实战：异步客户端与服务端
### 场景介绍
为了更直观地感受 Rust 异步编程在实际应用中的魅力，我们来构建一个简单的即时通讯场景。在这个场景中，客户端可以向服务端发送消息，服务端接收消息后，会将其广播给其他所有已连接的客户端，就像一个小型的在线聊天室 。通过这个示例，你将看到如何使用 Rust 的异步编程技术来实现高效的网络通信。

### 服务端实现
我们使用 Tokio 和 Warp 来构建这个异步服务端。

Tokio 是 Rust 中一个流行的异步运行时，它提供了异步 I/O、定时器等功能。

Warp 则是一个基于 Tokio 的 Web 框架，它使得构建 HTTP 服务器变得非常简单。

首先，在Cargo.toml文件中添加依赖：
```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
warp = "0.3"
tokio-tungstenite = "0.21"
futures-util = { version = "0.3.31" }
```

然后，编写服务端代码：
```rust
use std::collections::HashMap;
use std::sync::Arc;

use futures_util::{SinkExt, StreamExt};
use tokio::sync::{mpsc, RwLock};
use warp::{filters::ws::Message, Filter};

type ClientSender = mpsc::UnboundedSender<Message>;

struct ChatServer {
    clients: Arc<RwLock<HashMap<String, ClientSender>>>,
}

impl ChatServer {
    fn new() -> Self {
        ChatServer {
            clients: Arc::new(RwLock::new(HashMap::new())),
        }
    }

    async fn register(&self, name: String, sender: ClientSender) {
        let mut clients = self.clients.write().await;
        clients.insert(name, sender);
    }

    async fn unregister(&self, name: String) {
        let mut clients = self.clients.write().await;
        clients.remove(&name);
    }

    async fn broadcast(&self, message: Message) {
        let clients = self.clients.read().await;
        for (_, sender) in clients.iter() {
            if let Err(e) = sender.send(message.clone()) {
                eprintln!("Error sending message: {}", e);
            }
        }
    }
}

async fn handle_connection(
    server: Arc<ChatServer>,
    socket: warp::ws::WebSocket,
    name: String,
) {
    let (mut sender, mut receiver) = socket.split();
    let (tx, mut rx) = mpsc::unbounded_channel();

    server.register(name.clone(), tx).await;

    tokio::spawn(async move {
        while let Some(message) = receiver.next().await {
            if let Ok(msg) = message {
                if let Ok(text) = msg.to_str() {
                    let broadcast_msg = Message::text(format!("{}: {}", &name, text));
                    server.broadcast(broadcast_msg).await;
                }
            }
        }
        server.unregister(name.clone()).await;
        eprintln!("Client {} disconnected", name);
    });

    while let Some(message) = rx.recv().await {
        if let Err(e) = sender.send(message).await {
            eprintln!("Error sending message to client: {}", e);
            break;
        }
    }
}

#[tokio::main]
async fn main() {
    let server = Arc::new(ChatServer::new());
    let chat = warp::path!("chat" / String)
        .and(warp::ws())
        .map(move |name, ws: warp::ws::Ws| {
            let server = server.clone();
            ws.on_upgrade(move |socket| handle_connection(server, socket, name))
        });

    println!("Server running on http://127.0.0.1:3030");
    warp::serve(chat).run(([127, 0, 0, 1], 3030)).await;
}

```

在这段代码中，首先定义了一个ChatServer结构体，用于管理客户端连接和消息广播 。register方法用于将新的客户端加入到客户端列表中，unregister方法用于移除客户端，broadcast方法则用于将接收到的消息发送给所有已连接的客户端。

handle_connection函数处理每个客户端的连接，它负责接收客户端发送的消息，并将其广播给其他客户端 。

最后，在main函数中，创建了一个ChatServer实例，并使用 Warp 框架定义了一个路由，监听/chat/{name}路径的 WebSocket 连接 。

### 客户端实现
接下来，我们使用 Tokio 和tokio-tungstenite库来实现异步客户端。tokio-tungstenite是一个基于 Tungstenite 的异步 WebSocket 库，它与 Tokio 无缝集成，非常适合用于编写异步 WebSocket 客户端 。在Cargo.toml文件中添加依赖：
```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
warp = "0.3"
tokio-tungstenite = "0.21"
futures-util = { version = "0.3.31" }
```

然后，编写客户端代码：
```rust
use futures_util::{SinkExt, StreamExt};
use tokio::io::{self, AsyncBufReadExt};
use tokio_tungstenite::{connect_async, tungstenite::protocol::Message};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "ws://127.0.0.1:3030/chat/Alice";
    let (ws_stream, _) = connect_async(url).await?;

    // 使用 SinkExt::split() 拆分 Sink 和 Stream
    let (mut write, read) = ws_stream.split();

    // 创建消息通道
    let (tx, mut rx) = tokio::sync::mpsc::channel::<String>(10);

    // 发送任务：从通道接收消息并发送到 WebSocket
    tokio::spawn(async move {
        while let Some(message) = rx.recv().await {
            if let Err(e) = write.send(Message::Text(message)).await {
                eprintln!("发送错误: {}", e);
                break;
            }
        }
        // 显式关闭写端
        let _ = write.close().await;
    });

    // 接收任务：从 WebSocket 接收消息并打印
    tokio::spawn(async move {
        read.for_each(|message| async move {
            match message {
                Ok(Message::Text(text)) => println!("收到消息: {}", text),
                Ok(_) => println!("收到非文本消息"),
                Err(e) => eprintln!("接收错误: {}", e),
            }
        }).await;
    });

    // 异步读取标准输入
    let mut stdin = io::BufReader::new(io::stdin());
    loop {
        let mut input = String::new();
        stdin.read_line(&mut input).await?;
        let trimmed = input.trim().to_string();

        if trimmed == "exit" {
            break;
        }

        if let Err(e) = tx.send(trimmed).await {
            eprintln!("通道错误: {}", e);
            break;
        }
    }

    Ok(())
}
```

在这段代码中，首先使用connect_async函数连接到服务端。
然后，创建了一个通道tx和rx，用于发送和接收消息。

通过tokio::spawn创建了两个异步任务，一个用于从标准输入读取用户输入的消息，并通过通道发送给 WebSocket 连接；

另一个用于接收 WebSocket 连接上的消息，并打印到控制台。

最后，通过循环读取用户输入，当用户输入exit时，退出程序 。
### 运行与测试
在运行代码之前，请确保你已经安装了 Rust 环境。
- 启动服务端：
在终端中运行以下命令：
```shell
cargo run --bin server
```

- 启动客户端：
在另一个终端中运行客户端：
```shell
cargo run --bin client
```

当客户端连接到服务端后，你可以在客户端输入消息，服务端会将消息广播给所有已连接的客户端。

可以尝试打开多个客户端，同时发送和接收消息，感受 Rust 异步编程带来的高效并发体验。

例如，当客户端 1 发送 “Hello, Rust!” 时，客户端 2 和其他所有客户端都能即时收到这条消息，实现了实时的消息交互。

## Future、Executor 和 Waker：异步的幕后英雄
当我们深入 Rust 异步编程的核心，Future、Executor和Waker就像是幕后的英雄，默默支撑着整个异步体系的运转，它们各自扮演着独特而关键的角色，共同构建了高效的异步编程模型。

### Future 的工作原理
在 Rust 中，Future是一个代表异步计算的抽象，它定义了一个尚未完成的操作，这个操作在未来的某个时刻会产生一个结果。

Future的核心是poll方法，这个方法用于推进异步操作的执行。Future有两种状态：Ready和Pending。

当Future处于Ready状态时，表示异步操作已经完成，并且可以获取到结果；而当Future处于Pending状态时，则意味着异步操作还在进行中，尚未完成。

### Executor：任务调度者
Executor是负责调度Future执行的组件，它就像是一个任务调度者，管理和执行Future。

Executor的主要职责是通过不停调用Future的poll方法，直到完成。
### Waker：推进 Future 的执行
Waker是一个关键的概念，它用于通知Executor某个Future已经准备好，可以继续执行了。

当Executor调用某个Future的poll方法时，会生成一个Waker对象，并将其传递给Future。

如果此时Future还没有准备好，它会立即返回Pending状态，并将传入Waker保存起来。

当Future准备好后，它会调用Waker的wake方法，通知Executor这个Future已经准备好，可以继续执行了。

Executor会收到这个通知，然后再次调用这个Future的poll方法，继续推进Future的执行。

### 简单实现一个Future
下面是一个简单的Future实现，用于模拟一个异步任务的执行：
```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, Waker};

struct MyFuture {
    ready: bool,
    value: u32,
    waker: Option<Waker>,
}
impl MyFuture {
    fn new() -> Self {
        MyFuture { ready: false, value: 0, waker: None }           
    }
    fn async_run(&mut self) {
        self.ready = true;
        if let Some(waker) = self.waker.take() {
            waker.wake();
        }
    }
}
impl Future for MyFuture {
    type Output = u32;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.ready {
            Poll::Ready(self.value)
        } else {
            // 保存waker 稍后使用。
            self.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```
在这个示例中，MyFuture实现了Future trait，它有一个ready字段用于表示任务是否已经完成。在poll方法中，如果任务已经完成，返回Ready状态；否则，将ready设置为true，并返回Pending状态。
### 简单实现一个Executor
下面是一个简单的Executor实现，用于执行Future：
```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, Waker};
use waker_fn::waker_fn; // Cargo.toml: waker-fn = "1.1"
use futures_lite::pin; // Cargo.toml: futures-lite = "1.11"
use crossbeam::sync::Parker; // Cargo.toml: crossbeam = "0.8"

struct MyExecutor

impl MyExecutor {
    
    fn new() -> Self {
        MyExecutor

    }
    fn execute(&mut self, future: Pin<Box<dyn Future<Output = ()>>>) {
        let parker = Parker::new();
        let unparker = parker.unparker().clone();
        let waker = waker_fn(move || unparker.unpark());
        let mut context = Context::from_waker(&waker);
        loop {
            match future.as_mut().poll(&mut context) {
                Poll::Ready(value) => return value,
                Poll::Pending => parker.park()   
            }

        }
    }

}
```
在这个示例中，MyExecutor实现了一个简单的Executor。在execute方法中，通过循环调用Future的poll方法，直到任务完成。

在poll方法中，如果任务已经完成，返回Ready状态；否则，将Waker保存到waker字段中，并返回Pending状态。

这里用到的crossbeam crate的Parker 类型是一个简单的阻塞原语, 用于实现Waker：调用parker.park() 会阻塞当前的线程直到某个别的线程对相应的Unparker 调用.unpark()，而Unparker是 通过调用 parker.unparker() 获得。

最后，这个poll循环非常简单。传递一个携带我们的waker的上下文之后，我们poll future直到它返回Poll::Ready。如果它返回Poll::Pending，我们就park线程，它会阻塞直到waker 被调用。然后我们会再次尝试poll, 直到future返回Poll::Ready。
### 简单实现一个Waker
下面是一个简单的Waker实现，用于通知Executor某个Future已经准备好，可以继续执行了：
```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, Waker};
struct MyWaker {
    executor: Arc<MyExecutor>,

}
impl MyWaker {
    fn new(executor: Arc<MyExecutor>) -> Self {
        MyWaker { executor }
    }

}
impl Waker for MyWaker {
    fn wake(&self) {
        self.executor.wake();
    }

}
```
在这个示例中，MyWaker实现了Waker trait，它有一个executor字段用于保存Executor对象。

在wake方法中，调用Executor的wake方法，通知Executor某个Future已经准备好，可以继续执行了。

## Pin：解决自引用问题的利器
在Future的poll方法中，self类型并不是Future，而是Pin<&mut Self>。

这是因为Future trait是一个自引用的trait，当Future第一次被poll后。异步函数体开始执行，异步过程中就可能借用存储在future中的变量的引用，然后await，导致future的一部分被借用。

这样在future第一次被poll之后，Future就已经不能安全地move了。

如果假设允许Future被move，那么异步过程中对Future的引用就会出现悬垂引用，这是不安全的。

为了解决这个问题，Rust引入了Pin类型，它允许我们将一个可变引用Pin到一个位置，从而确保这个引用在异步过程中不会被move。

Pin类型的定义如下：
```rust
pub struct Pin<Ptr> {
    pub __pointer: Ptr,
}
```
Pin 类型是一个future的指针的包装，它限制了指针的用法来确保future一旦被poll之后就不能再move。

Pin<&mut T> 和Pin<Box<T>> 是典型的pinned指针

每一种获得future的pinned指针的方式都意味着放弃future的所有权，并且没有办法将它转变回来。

pinned指针自身可以move到任何地方，但move一个指针不会move它引用的对象。

因此创建一个pinned指针的过程就是在证明你永久放弃了move这个future的能力

一旦Pin了一个future，如果你想poll它，所有 Pin<pointer to T> 类型有一个as_mut 方法解引用指针并返回一个Pin<&mut T>，这正是poll 所需的。

as_mut 方法还可以帮助你在不放弃所有权的情况下poll一个future

### UnPin
UnPin 是一个trait，用来标记一个类型是UnPin的类型。
```rust
pub trait Unpin {}
```
当一个类型没有必要使用Pin时，它就可以实现UnPin trait。
Rust中几乎所有的类型都通过编译器的特殊支持实现了Unpin。异步函数和块的future是这个规则的例外。

对于Unpin 类型，Pin 不会增加任何限制。可以使用Pin::new 从一个普通的指针制造一个pinned指针，也可以使用Pin::into_inner 转换回指针


例如，String 实现了Unpin，因此我们可以写：
```rust
let mut string = "Pined?".to_string();
let mut pinned: Pin<&mut String> = Pin::new(&mut string);
pinned.push_str(" Not");
Pin::into_inner(pinned).push_str(" so much.");
let new_home = string;
assert_eq!(new_home, "Pinned? Not so much.");
```

即使构造出一个Pin<&mut String> 之后，仍然有对原本的字符串的完整可变访问权限，并且一旦Pin 被into_inner 消耗，可变引用就会消失，此时我们还可以把它move进一个新的变量中。

因此对于那些是Unpin 的类型——几乎所有类型都是——Pin 只是一个该类型指针的无效的包装。

