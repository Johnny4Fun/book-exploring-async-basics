# Node是什么？

首先我们简要解释一下：Node是什么，以保证我们对此有一致的认识。

Node是一个JavaScript的运行时，能够让你在桌面端（或者服务端）运行JavaScript。JavaScript的设计初衷是一门在浏览器中运行的脚本语言，这就意味着它必须要依赖浏览器进行解释编译、提供运行时。

桌面端需要提供解释（或者编译）以及运行时，才能让JavaScript代码有效地运行起来。而在桌面端，[V8 javascript 引擎](https://en.wikipedia.org/wiki/V8_JavaScript_engine) 提供了编译Javascript的能力，[Node](https://en.wikipedia.org/wiki/Node.js) 则提供了运行时环境。

从语言设计的角度来看，JavaScript有一个优点：在JavaScript的世界里，一切都是可以被异步处理。正如你所知，当我们想要最大程度地利用硬件资源，尤其是需要处理大量的I/O操作时，异步处理是相当重要的一个特性。

一个典型的场景就是Web服务器。Web服务器需要处理大量的I/O任务——要么是从文件系统中读取内容，要么是通过网卡与客户端进行通信。

## 为什么是Node？

- 当在浏览器上进行web开发时，JavaScript是必不可少的。如果服务端也使用JavaScript，就能将前后端开发的语言统一为JavaScript了。
- 在后端和前端之间存在代码重用的可能性
- Node的设计使其可以成为非常高性能的web服务器
- 当你只使用JavaScript时，处理json、使用web API是非常简单的。

## 几点有用的真相

我们先揭开几个概念的神秘面纱，这样一来我们在开始编写代码时能更好地跟上思路。

### JavaScript的事件循环

JavaScript只是一门单纯的脚本语言，单靠语言本身发挥不出太大的作用，本身也没有事件循环。而Web浏览器会提供一个运行时环境，其中就包括了一个事件循环。而在服务端，事件循环则由Node提供。

可以这么说，JavaScript作为一门编程语言（由于其基于回调的模型），如果没有某种事件循环的支持，就无法运行，但这并不是重点。

### Node是多线程的

和我之前好几次看到的说法相反，Node使用了一个线程池，所以它是多线程的。然而，Node中推动代码执行的部分，确实是运行在单个线程上的。当我们说“不要阻塞事件循环”时，实际上，我们所指的就是这个线程，因为那样会使得Node无法继续推动其他任务的执行。

我们之后会更加清楚地理解：为什么阻塞这个线程会出现问题，以及我们如何处理这个问题。

### V8引擎

这部分是我们需要重点关注的。V8引擎是一种JavaScript的JIT编译器（Just-In-Time Compiler）。这就说明，当你写了一个`for`循环，V8引擎会将其翻译成能够运行在CPU上的指令。JavaScript的引擎有很多，而Node最初选择基于V8引擎实现。

单单V8引擎本身对我们而言没什么用；它只是在解释我们的JavaScript代码。它无法执行I/O、创建运行时或者其他类似的事情。仅仅靠V8引擎，JavaScript代码所能完成的工作非常有限。

> 因为我们使用的是Rust（尽管我们想让代码看起来像JavaScript），所以我们不会涉及解释JavaScript的内容。我们只需要专注于Node是如何工作的以及如何处理并发即可。

### Node的事件循环

Node内部将实际工作分成了两类：

#### I/O密集型任务

主要等待某一外部事件发生的任务，将由跨平台的`libuv`（在我们的例子中使用的是`minimio`）来处理。这二者都是基于epoll/kqueue/IOCP实现的跨平台事件队列。

#### CPU密集型任务

CPU密集型任务交由线程池来处理。线程池的默认大小为4，可以通过Node的运行时进行配置。

无法使用跨平台事件队列来处理的I/O任务（比如说我们在例子中所出现的读取文件）也可以用线程池处理。

Node中大多数的C++扩展会使用这个线程池来执行工作。这也正是计算密集型任务使用这些C++扩展的原因之一。

## 更多的信息

如果你想知道更多关于Node的事件循环的内容，我推荐一个`libuv`的文档，以及两个相关的演讲。

[Libuv Design Overview](http://docs.libuv.org/en/v1.x/design.html#design-overview)（译者也推荐！）

[第一个视频](https://www.youtube.com/embed/PNa9OMajw9w)是来自[@piscisaureus](https://github.com/piscisaureus)一场精彩的15分钟演讲。我特别推荐这段简短切题的演讲。

[第二个视频](https://www.youtube.com/embed/zphcsoSJMvM)相对较长，但也很精彩，来自[Bryan Hughes](https://github.com/nebrius)。

现在，放松点，喝杯茶，坐下来，我们将一起仔细地过一遍这一切。

