# 实现我们的运行时

让我们开始写一些代码吧；我们有很多事情要做呢。

我将用我认为最容易理解的方式来解释这一切。这也意味着有时我会提前引入部分代码，稍后解释会这些代码。如果你一开始无法理解一些内容，不要担心，之后我会把所有的东西都过一遍。

首先，我们需要创建一个新Rust项目：

```
cargo new async-basics
cd async-basics
```

正如我之前提到的，我们会使用到`minimio`库（在另一本里会具体解释，但是如果你感兴趣的话，可以浏览源码）：

在文件`Cargo.toml`中

```toml
[dependencies]
minimio = {git = "https://github.com/cfsamson/examples-minimio", branch = "master"}
```

另一个选择就是直接克隆包含所有代码的repo，通过命令：

```
git clone https://github.com/cfsamson/examples-node-eventloop
```

接着，我们需要一个`运行时`结构体/对象来承载所有我们`运行时`所需要记录的状态。

首先，打开`main.rs`（位于`src/main.rc`）。

> 我们大致按照在本书中介绍的顺序，把所有代码都集中在一个文件里。

## Runtime 结构体

我在代码中添加了注释，这样更容易记忆和理解。

```rust, no_run
pub struct Runtime {
    /// Available threads for the threadpool
    /// 线程池可用的线程
    available_threads: Vec<usize>,
    /// Callbacks scheduled to run
    /// 预定执行的回调函数
    callbacks_to_run: Vec<(usize, Js)>,
    /// All registered callbacks
    /// 注册过的所有回调函数
    callback_queue: HashMap<usize, Box<dyn FnOnce(Js)>>,
    /// Number of pending epoll events, only used by us to print for this example
    /// 等待中的epoll事件的数量，在这个例子中只是我们用来打印信息的
    epoll_pending_events: usize,
    /// Our event registrator which registers interest in events with the OS
    /// 事件注册器，用于向操作系统注册对事件的兴趣
    epoll_registrator: minimio::Registrator,
    /// The handle to our epoll thread
    /// epoll线程的句柄
    epoll_thread: thread::JoinHandle<()>,
    /// None = infinite, Some(n) = timeout in n ms, Some(0) = immediate
    /// None = 无限长, Some(n) = 在n毫秒后超时, Some(0) = 立即
    epoll_timeout: Arc<Mutex<Option<i32>>>,
    /// Channel used by both our threadpool and our epoll thread to send events
    /// to the main loop
    /// 管道，线程池和epoll线程通过管道向主事件循环发送事件
    event_reciever: Receiver<PollEvent>,
    /// Creates an unique identity for our callbacks
    /// 为回调创建唯一标识
    identity_token: usize,
    /// The number of events pending. When this is zero, we're done
    /// 待执行的事件数量。当数量为0时，程序结束
    pending_events: usize,
    /// Handles to our threads in the threadpool
    /// 线程池中的线程对应的句柄
    thread_pool: Vec<NodeThread>,
    /// Holds all our timers, and an Id for the callback to run once they expire
    /// 保存所有的定时器和对应回调的id
    timers: BTreeMap<Instant, usize>,
    /// A struct to temporarely hold timers to remove. We let Runtinme have
    /// ownership so we can reuse the same memory
    /// 临时保存需要移除的定时器。让运行时持有它们，这样我们就可以复用这些定时器
    timers_to_remove: Vec<Instant>,
}
```

在之后的章节中，我们会一一介绍结构体里的每一个成员。

接下来，定义一些会用到的类型。

## Task

```rust, no_run
struct Task {
    task: Box<dyn Fn() -> Js + Send + 'static>,
    callback_id: usize,
    kind: ThreadPoolTaskKind,
}

impl Task {
    fn close() -> Self {
        Task {
            task: Box::new(|| Js::Undefined),
            callback_id: 0,
            kind: ThreadPoolTaskKind::Close,
        }
    }
}
```
我们需要一种任务对象，用来表示我们希望在线程池中完成的任务。在[之后的章节](./7_9_infrastructure.md) 中我会详细地介绍这个类型。所以如果你觉得这很难理解，不要太担心，后面我都会解释的。

我们还实现了一个`Close`任务。在结束时，我们通过投递该任务从而指示程序清理和关闭线程池。

`|| Js::Undefined`看起来可能有点奇怪，其实这只是一个返回`Js::Undefined`的函数。因为我们没必要仅仅为了这个情况就把`task`设置为`Option`。

这样我们就不需要在面对`task`时使用`match`和`map`了。

## NodeThread

`NodeThread`代表线程池中的一个线程。正如你们所见，这其中包含`JoinHandle`（当我们调用`thread::spawn`时会返回的对象）和一个管道的发送端。这个管道传输`Event`类型的消息。

```rust, no_run
#[derive(Debug)]
struct NodeThread {
    pub(crate) handle: JoinHandle<()>,
    sender: Sender<Task>,
}
```

这里我们引入了两个新的类型：`Js`和`ThreadPoolTaskKind`。我们先讲`ThreadPoolTaskKind`。

在我们的例子里，一共有三种类型的事件：一个`FileRead`事件代表一个文件被读取，一个`Encrypt`事件代表`Crypto`模块中的一个操作。一个`Close`事件用于在关闭事件循环放出时`threadpool`中的线程，并且在进程退出前结束这些线程。

## ThreadPoolTaskKind

正如你所理解的那样，这个对象只会在`线程池`中使用。

```rust,no_run
pub enum ThreadPoolTaskKind {
    FileRead,
    Encrypt,
    Close,
}
```

## Js

接下来则是`Js`对象。它代表了不同的`JavaScript`类型，只是用来让我们的代码看起来更”JavaScripty“，但同时也便于我们对闭包的返回类型进行抽象。

我们将在这个对象上实现两个简便方法，从而使得”JavaScripty“的代码看起来更加清晰。

根据模块文档，我们能知道返回的各种类型——就像当你使用Node模块时会从文档中知道的那样，但是，在Rust中我们还是处理这些类型，所以这些做法会让我们更加轻松。

```rust,no_run
#[derive(Debug)]
pub enum Js {
    Undefined,
    String(String),
    Int(usize),
}

impl Js {
    /// Convenience method since we know the types
    /// 简便方法，因为已知具体类型
    fn into_string(self) -> Option<String> {
        match self {
            Js::String(s) => Some(s),
            _ => None,
        }
    }

    /// Convenience method since we know the types
    /// 简便方法，因为已知具体类型
    fn into_int(self) -> Option<usize> {
        match self {
            Js::Int(n) => Some(n),
            _ => None,
        }
    }
}
```

## PollEvent

接下来，轮到`PollEvent`。我们定义一个`enum`表示我们可以发送到的`eventpool`的事件类型，从而可以表示从`基于epoll`的事件队列和`threadpool`中接收到的事件。

```rust
/// Describes the three main events our epoll-eventloop handles
/// 描述了Epoll事件循环能够处理的三种事件
enum PollEvent {
    /// An event from the `threadpool` with a tuple containing the `thread id`,
    /// the `callback_id` and the data which the we expect to process in our
    /// callback
    /// 来自`线程池`的事件，持有一个元组，包含`thread id`，`callback_id`和我们希望传入回调的数据
    Threadpool((usize, usize, Js)),
    /// An event from the epoll-based eventloop holding the `event_id` for the
    /// event
    /// 基于epoll的事件循环所产生的事件，持有对应事件的`event_id`
    Epoll(usize),
    Timeout,
}
```

## const RUNTIME

最后，我们还有其他的便利设施，能够让我们的代码看起来更像JavaScript。

首先我们定义了一个指代`Runtime`静态变量。它其实是一个指向`运行时`的指针，一开始会被初始化为一个空指针。

我们需要使用`unsafe`来修改这个静态变量。之后我会解释为什么这个操作是安全的，同时我想提一下，其实通过使用[lazy_static](https://github.com/rust-lang-nursery/lazy-static.rs)包，我们可以避免这种情况。但是这样既需要引入对`lazy_static`的依赖，又会使得代码的可读性下降，而且`lazy_static`相当复杂。


```rust, no_run
static mut RUNTIME: *mut Runtime = std::ptr::null_mut();
```

接下来，我们来看看这个运行时的核心部分——主循环

## 下一部分

现在我们已经在这一章中介绍了运行时的大部分内容。下一章将着重于实现我们需要的所有功能。
