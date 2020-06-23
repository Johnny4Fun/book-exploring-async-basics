# 创建运行时

## 线程池

我们还没有实现线程池和I/O事件循环。所以这就是接下来我们的任务。

### 一步一步来

准备好了就开始吧！

第一件事，给我们的`Runtime`类添加一个`new`方法，使其能够返回一个类的实例。

首先是线程池。我们要做的第一件事就是创建一个管道（channel），为线程池中的线程提供一个给主线程传递消息的途径。

这个管道会接收一个元组`(usize, usize, Js)`，代表了`thread_id`、`callback_id`和当运行`Task`时返回的数据。

`Receiver`端会被储存在`Runtime`中，而`Sender`端则会被拷贝到每一个线程中。

我们借鉴Node默认4个线程的做法。在`Node`中线程数是可配置的，而我们偷个懒，就直接使用硬编码。

```rust, no_run
let (event_sender, event_reciever) = channel::<PollEvent>();
let mut threads = Vec::with_capacity(4);

for i in 0..4 {
    let (evt_sender, evt_reciever) = channel::<Task>();
    let event_sender = event_sender.clone();

    let handle = thread::Builder::new()
        .name(format!("pool{}", i))
        .spawn(move || {

            while let Ok(task) = evt_reciever.recv() {
                print(format!("received a task of type: {}", task.kind));

                if let ThreadPoolTaskKind::Close = task.kind {
                    break;
                };

                let res = (task.task)();
                print(format!("finished running a task of type: {}.", task.kind));

                let event = PollEvent::Threadpool((i, task.callback_id, res));
                event_sender.send(event).expect("threadpool");
            }
        })
        .expect("Couldn't initialize thread pool.");

    let node_thread = NodeThread {
        handle,
        sender: evt_sender,
    };

    threads.push(node_thread);
}

```

接着就是创建线程。`for i in 0..4`就是在值0、1、2、3上遍历。因为我们会把每一个线程都添加进一个`Vec`中，所以这四个值既是线程的id，也是他们在`Vec`中对应的序号。

接下来，我们为每个线程都创建一个新的管道，主线程使用这些管道给线程池中的子线程发送消息。每个线程都会持有自己的`Receiver`，而我们会把`Send`端存储在结构体`NodeThread`中，这个结构体用来表示线程池中的线程。

```rust, no_run
let (evt_sender, evt_reciever) = channel::<Event>();
let event_sender = event_sender.clone();
```
正如你所看到的，我们把`Sender`端复制、传递给每一个线程，以便它们可以给`main`线程发送消息。（注意区分event_sender和evt_sender，前者是线程池中每个线程各持有一个发送端，主线程持有接收端；后者是每个线程都有一个对应管道的接收端，主线程持有所有的发送端，用于发送任务)。

这些工作就绪后，因为要给每个线程设置名称，所以我们会使用`thread::Builder::new()`而非`thread::spawn`。我们在事件`print`时会使用到这个名称，以便更加清晰地展示打印信息的线程来源。

```rust, no_run
let handle = thread::Builder::new()
        .name(format!("pool{}", i))
        .spawn(move || {
```
最终，我们使用`spawn`创建线程，也创建了一个闭包（Closure）。

> 为什么在这个闭包中，我们要使用`move`关键字？
>
> 原因就是闭包是主线程创建的，所以所有包含的环境变量都需要有拥有权，因为闭包不能引用`main`线程栈空间上的任何值。在这里我引用[chapter about `closures` in TRPL](https://doc.rust-lang.org/1.30.0/book/first-edition/closures.html#closures)中的一段相关内容。
>
> __…他们赋予闭包单独的栈空间。如果不使用move，闭包可能会绑定到创建它的那个线程的堆栈，而move闭包是自包含的。__

子线程的逻辑相当简单，大部分代码是用来打印给信息，供我们查看的。

```rust, no_run
while let Ok(task) = evt_reciever.recv() {
        print(format!("received a task of type: {}", task.kind));

        if let ThreadPoolTaskKind::Close = task.kind {
            break;
        };

        let res = (task.task)();
        print(format!("finished running a task of type: {}.", task.kind));

        let event = PollEvent::Threadpool((i, task.callback_id, res));
        event_sender.send(event).expect("threadpool");
    }
})
```

首先，我们需要监听管道的`Receive`端（`Send`端由`main`线程持有）。这个函数会使该线程`暂停（park）`，直至我们收到一条消息，所以当处于等待状态时，管道中没有可以消费的消息。

当我们获取到一个`task`时，我们首先打印其任务类型。

接着，需要判断这个任务是否是`Close`类型，如果是，则直接跳出事件循环，之后该线程就会关闭。

如果不是`Close`类型的任务，我们就会执行该任务`let res= (task.task)();`。这里就是任务真正被执行的位置。从函数签名可以看出，任务执行完成后返回一个`Js`对象。

接着，我们输出信息：任务执行完毕。然后，我们将投递一个携带`thread_id`、`callback_id`和`返回信息的Js对象` 的`PollEvent::Threadpool`类型事件至`main`线程。

再次回到`main`线程，我们将`JoinHandle`和管道的`Send`端都存储在`NodeThread`结构体中，然后把它推送至存储线程的容器（这就代表线程池）中。

## Epoll线程

这部分是关于Epoll/Kqueue/IOCP线程的处理。该线程只会等待操作系统上报的事件，一旦有事件完成，它会把事件的Id发送至主线程，主线程再去处理事件和调用回调函数。

这部分代码有点晦涩，但是我们会一步一步讲解的。

代码：

```rust, no_run
let mut poll = minimio::Poll::new().expect("Error creating epoll queue");
let registrator = poll.registrator();
let epoll_timeout = Arc::new(Mutex::new(None));
let epoll_timeout_clone = epoll_timeout.clone();

let epoll_thread = thread::Builder::new()
    .name("epoll".to_string())
    .spawn(move || {
        let mut events = minimio::Events::with_capacity(1024);

        loop {
            let epoll_timeout_handle = epoll_timeout_clone.lock().unwrap();
            let timeout = *epoll_timeout_handle;
            drop(epoll_timeout_handle);

            match poll.poll(&mut events, timeout) {
                Ok(v) if v > 0 => {
                    for i in 0..v {
                        let event = events.get_mut(i).expect("No events in event list.");
                        print(format!("epoll event {} is ready", event.id().value()));

                        let event = PollEvent::Epoll(event.id().value() as usize);
                        event_sender.send(event).expect("epoll event");
                    }
                }
                Ok(v) if v == 0 => {
                    print("epoll event timeout is ready");
                    event_sender.send(PollEvent::Timeout).expect("epoll timeout");
                }
                Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {
                    print("received event of type: Close");
                    break;
                }
                Err(e) => panic!("{:?}", e),
                _ => unreachable!(),
            }
        }
    })
    .expect("Error creating epoll thread");
```

从初始化变量开始：

```rust, no_run
let mut poll = minimio::Poll::new().expect("Error creating epoll queue");
let registrator = poll.registrator();
let epoll_timeout = Arc::new(Mutex::new(None));
let epoll_timeout_clone = epoll_timeout.clone();
```

首先，我们初始化一个全新的`minimio::Poll`对象。它是`kqueue/epoll/IOCP`事件队列的主入口。

> “minimio::Poll”在幕后做了几件事。它为我们建立了一个结构来存储一些关于事件队列状态的信息，最重要的是对底层操作系统进行系统调用，并向操作系统申请一个`epoll`实例或`kqueue`或一个`完成端口`的句柄。我们不会在这里注册任何事件，但是我们需要使用这个句柄，确保接下来我们能够向事件队列注册感兴趣的事件。

接着是`minimio`的一部分 ：获取一个`Registrator`。这个结构体是`Poll`结构体中”分离“出来的，但是它也拥有一个事件队列**句柄的拷贝**。

通过这种方式，我们可以将`Registrator`存储在主线程中，并将`Poll`实例传递给`epoll`线程。Registrator只能向事件队列注册对事件的兴趣，仅此而已。

> `Registrator`如何知道`epoll`线程没有停止呢？
>
> 我们会在下一本书中详细讨论这个问题。简单来说，`Poll`和`Registrator`都持有一个`AtomicBool` 的引用，用来指示队列是否还”活着“。在`Poll`的`Drop`方法，会将这个标识位设置为false，此时，如果调用函数注册事件，会返回一个`Err`。

`epoll_timeout` 是距离下次定时器超时的时间。如果没有定时器，那么该值会被设置为`Node`。我们将这个值包装在`Arc<Mutex<>>`，因为我们会在main线程中修改这个变量的值，在`epoll`线程中读取这个变量的值。

`epoll_timeout_clone` 就是增加`Arc`的引用计数，以便我们将其传入`epoll`线程。

接下来就是创建线程。和之前线程池的做法一样，只是这次我们将这个线程命名为`epoll`。

```rust, no_run
let epoll_thread = thread::Builder::new()
    .name("epoll".to_string())
    .spawn(move || {
```

接下来，我们进入`epoll`线程内部，并定义这个线程如何轮询和处理事件。

首先，我们分配一个缓冲区来保存从`poll`实例中获取的事件对象。这些对象包含了发生事件的信息，包括我们在注册事件时传入的`token`。这个`token`标识发生了哪个事件。在我们的示例中，token是一个简单的`usize`。

```rust, no_run
let mut events = minimio::Events::with_capacity(1024);
```
我们在此处分配缓冲区是因为我们在执行时只分配一次缓冲区，并且我们希望避免在`loop`的每一次轮都分配一个新的缓冲区。

概括而言，我们的`epoll`线程将运行一个循环，该循环有意识地轮询是否新事件发生。

```rust, no_run
loop {
```

循环内部有一个比较有趣的逻辑——首先我们读取超时时间的值，而它应该与主循环中的下一个（最先发生的）定时器的超时时间点一致。

```rust, no_run
let epoll_timeout_handle = epoll_timeout_clone.lock().unwrap();
let timeout = *epoll_timeout_handle;
drop(epoll_timeout_handle);
```

为了实现这一点，我们首先需要`lock`互斥锁mutex，这样我们就拥有对`timeout`值的独占访问权。现在，`epoll_timeout_handle`的类型是 `Option<i32>` ，因为`i32 `实现了`Copy`特性，所以我们能够直接对`epoll_timeout_handle`解引用，而这个值会被拷贝、存储至`timeout`变量中。

`drop(epoll_timeout_handle)`这样的操作并不常见。我们调用`epoll_timeout_clone.lock().unwrap()`返回的是`MutexGuard`，是一种[RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization)的guard。这意味着它会一直持有一种资源（此处是对mutex的锁）直至它被销毁（释放）。在Rust中，值被`Dropped`，就等同于释放；而释放通常来说是发生在作用域结束时（`{...}`。

我们必须释放这个锁，因为接下来的函数调用一直阻塞，直至有事件发生，这意味着当main线程尝试在`epoll_timeout`写入值时，这个锁就还没有被释放，最终导致`死锁（deadlock）`。

接下来这部分很棘手，但是请记住，此处的大部分工作只是为了打印信息便于观察。

调用`poll`会阻塞该循环，直到有事件发生**或者**到达超时时间。`poll`接收我们之前分配的缓冲区的独占引用，以及超时时间`Option<i32>`。`None`值表示无期限阻塞。

> 当我们提到`block`时，我们想表达的是：操作系统会挂起我们的线程，然后切换至其他线程的上下文。然而，操作系统依然会跟踪 `epoll`线程，监听事件，并且当我们注册的感兴趣的事件发生时，它会再次唤醒`epoll`线程。

```rust, no_run
match poll.poll(&mut events, timeout) {
    Ok(v) if v > 0 => {
        for i in 0..v {
            let event = events.get_mut(i).expect("No events in event list.");
            print(format!("epoll event {} is ready", event.id().value()));

            let event = PollEvent::Epoll(event.id().value() as usize);
            event_sender.send(event).expect("epoll event");
        }
    }
    Ok(v) if v == 0 => {
        print("epoll event timeout is ready");
        event_sender.send(PollEvent::Timeout).expect("epoll timeout");
    }
    Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {
        print("received event of type: Close");
        break;
    }
    Err(e) => panic!("{:?}", e),
    _ => unreachable!(),
}
```

我们会对`poll`的返回值进行`match`，所以当操作系统返回（唤醒线程）时，我们能够决定接下来要做什么。

大致有4种情形：

1. 返回值为Ok(n) 且n大于0，这种情况就表示我们有事件需要去处理
2. 返回值为Ok(n)且n等于0，那么我们就能知道，这要么是一次虚假唤醒，要么是poll超时了
3. 返回值为Err且类型为`Interrupted`，这种情况下，我们把这作为一种关闭信号，停止循环
4. 返回值为Err但不是`Interrupted`类型，我们就知道大事不妙了，并且触发`panic!`

如果你之前没有见过`Ok(v) if v > 0`这样的语法，那么我现在告诉你：我们把这叫做`match guard`，这种语法使我们能够精细化匹配。在这里，我们只匹配`v`值大于`0`的情况。

为了完整起见，我再解释一下`Err(ref e) if e.kind()`：`ref`关键字是告知编译器我们只需要`e`的引用，并不需要取得其所有权。

最后的情况`_ => unreachable!()`是因为编译器并不知道我们已经覆盖了所有`Ok()`的可能情况。这个值的类型是`Ok(usize)`，所以它不可能为负数，所以这个分支的作用就是告诉编译器：我们已经覆盖了所有可能情况。

最后我们创建`Runtime`结构体，存入之前我们初始化的所有数据：

```rust, no_run
  Runtime {
    available_threads: (0..4).collect(),
    callbacks_to_run: vec![],
    callback_queue: HashMap::new(),
    epoll_pending_events: 0,
    epoll_registrator: registrator,
    epoll_thread,
    epoll_timeout,
    event_reciever,
    identity_token: 0,
    pending_events: 0,
    thread_pool: threads,
    timers: BTreeMap::new(),
    timers_to_remove: vec![],
}
```

值得注意的是，因为我们已知所有的线程都是可用的，所以`(0..4).collect()`就是创建一个内容为`[0, 1, 2, 3]`的`Vec<usize>`对象。

在Rust中，当我们这样写…

```rust
...
epoll_registrator: registrator,
epoll_thread,
...
```

…我们就是将`registrator`赋值给`epoll_registrator`。而由于我们已经有一个名为`epoll_thread`的变量，所以我们不需要写成`epoll_thread: epoll_thread`，因为编译器会帮我们处理这种情况。

目前，最终的runtime初始化代码违背了所有“最佳实践”所倡导的原则——将函数的长度控制在xx行以内，但是对于我们的例子来说，我认为如果我们将所有的逻辑按时间顺序全部写到一起，而且不需要在函数中频繁地跳转，这样会更加简单明了。

```rust
impl Runtime {
    pub fn new() -> Self {
        // ===== THE REGULAR THREADPOOL =====
        let (event_sender, event_reciever) = channel::<PollEvent>();
        let mut threads = Vec::with_capacity(4);

        for i in 0..4 {
            let (evt_sender, evt_reciever) = channel::<Task>();
            let event_sender = event_sender.clone();

            let handle = thread::Builder::new()
                .name(format!("pool{}", i))
                .spawn(move || {

                    while let Ok(task) = evt_reciever.recv() {
                        print(format!("received a task of type: {}", task.kind));

                        if let ThreadPoolTaskKind::Close = task.kind {
                            break;
                        };

                        let res = (task.task)();
                        print(format!("finished running a task of type: {}.", task.kind));

                        let event = PollEvent::Threadpool((i, task.callback_id, res));
                        event_sender.send(event).expect("threadpool");
                    }
                })
                .expect("Couldn't initialize thread pool.");

            let node_thread = NodeThread {
                handle,
                sender: evt_sender,
            };

            threads.push(node_thread);
        }

        // ===== EPOLL THREAD =====
        let mut poll = minimio::Poll::new().expect("Error creating epoll queue");
        let registrator = poll.registrator();
        let epoll_timeout = Arc::new(Mutex::new(None));
        let epoll_timeout_clone = epoll_timeout.clone();

        let epoll_thread = thread::Builder::new()
            .name("epoll".to_string())
            .spawn(move || {
                let mut events = minimio::Events::with_capacity(1024);

                loop {
                    let epoll_timeout_handle = epoll_timeout_clone.lock().unwrap();
                    let timeout = *epoll_timeout_handle;
                    drop(epoll_timeout_handle);

                    match poll.poll(&mut events, timeout) {
                        Ok(v) if v > 0 => {
                            for i in 0..v {
                                let event = events.get_mut(i).expect("No events in event list.");
                                print(format!("epoll event {} is ready", event.id().value()));

                                let event = PollEvent::Epoll(event.id().value() as usize);
                                event_sender.send(event).expect("epoll event");
                            }
                        }
                        Ok(v) if v == 0 => {
                            print("epoll event timeout is ready");
                            event_sender.send(PollEvent::Timeout).expect("epoll timeout");
                        }
                        Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {
                            print("received event of type: Close");
                            break;
                        }
                        Err(e) => panic!("{:?}", e),
                        _ => unreachable!(),
                    }
                }
            })
            .expect("Error creating epoll thread");

        Runtime {
            available_threads: (0..4).collect(),
            callbacks_to_run: vec![],
            callback_queue: HashMap::new(),
            epoll_pending_events: 0,
            epoll_registrator: registrator,
            epoll_thread,
            epoll_timeout,
            event_reciever,
            identity_token: 0,
            pending_events: 0,
            thread_pool: threads,
            timers: BTreeMap::new(),
            timers_to_remove: vec![],
        }
    }
}
```
