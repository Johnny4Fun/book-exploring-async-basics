# 主循环

我们把事件循环的逻辑都放在`Runtime`类中的`run`方法里。在本章所展示的代码就是`run`方法的主体。

`Runtime`类中的`run`函数会销毁`self`，所以我们需要在最后调用`Runtime`实例的`run`函数。

在末尾处我会把整个方法都贴出来，方便浏览。

```rust, no_run
impl Runtime {
    pub fn run(mut self, f: impl Fn()) {
        ...
    }
}
```

## 初始化


```rust, no_run
let rt_ptr: *mut Runtime = &mut self;
unsafe { RUNTIME = rt_ptr };
let mut ticks = 0; // just for us priting out

// First we run our "main" function
f();
```

我们在前两行使用了一个`黑魔法`，使我们的代码会“看起来”更像JavaScript。取self的指针，再将其赋值给全局变量`RUNTIME`。

当然我们也可以把`runtime`作为参数在需要的时候传递，但是那样太冗余了。另一个选项是使用`lazy_static`包来初始化`RUNTIME`，这种方式会更加安全，但我们就需要解释`lazy_static`做了什么，以维持目标：最大限度地降低神秘感。

事实上，我们只会对这个变量赋值一次，而且赋值发生在开始启动事件循环，并且我们只会在创建这个对象的线程中使用它，所以尽管看起来不太美观，但它是安全的。

`ticks`仅仅是一个计数器，用于跟踪和展示循环进行的次数。

初始化的最后一步是我们通过调用`f()`来启动一切。`f`就是我们在上一节中提到的函数`javascript`。所以如果f为空的话，那么啥都不会发生。

## 启动事件循环

```rust, no_run
// ===== EVENT LOOP =====
// ===== 事 件 循 环 =====
while self.pending_events > 0 {
    ticks += 1;
```

`self.pending_events`负责记录挂起事件的数量，这样当没有事件剩余，意味着事件循环已经完成，我们就可以退出循环。

那这些事件从哪里产生的呢？在之前的章节[Introducing our main example](./7_0_introducing_our_main_example.md)引入的名为`javascript`的函数，也就是`f`中，你可能已经注意到了，我们调用了诸如`set_timeout`和`Fs::read`一类的函数。而这些函数是Node的运行时所定义的（我们的运行时也定义了类似的函数），他们的主要作用就是创建任务、注册对事件的兴趣。当注册任务或者兴趣时，这个计数器`self.pending_events`的值会增加；

`ticks`仅仅就是计数器加一。

## 1. 处理定时器

`self.process_expired_timers();`

这个方法会检查当前是否有到期的定时器。如果发现有到期的定时器，那么我们就需要调度定时器绑定的回调函数，使其在下次调用`self.run_callbacks()`执行。

值得注意的是，超时时间设置为0的定时器在我们进入`self.process_expired_timers`函数时已经超时，所以对应的回调也会被执行。

## 2. 回调函数

`self.run_callbacks();`

其实我们本可以选择在处理定时器的`阶段`就执行所有到期定时器的回调的，但是我们接下来的步骤正好就是执行回调函数，所以我们就在这步执行了。

> 这一步出现在此处似乎并不是很有必要，但是在Node中，确实存在这样的一个函数。某些类型的回调会被推迟到事件循环的下一轮才执行，也就是说，这类回调不会在超时后立即执行。虽然我们不会在这个例子中实现该功能，但是这一点值得一提。

## 3. 空转/准备

这一步是Node内部实现中最常用的一步。虽然这一步对于我们理解全局影响不大，但是因为你会在Node的文档中看到这一点，所以我把这一部分包括进来，会便于你能明白当前这一步在主循环中的位置。

## 4. Poll

这是一个关键步骤。在这一步中，我们会从线程池或`epoll`事件队列中接收事件。

当我提到`epoll`时，我指的就是`epoll/kqueue/IOCP`事件队列，也就是说，我们等待的并不局限于`epoll`这个事件队列。从现在开始，为了简洁起见，我将在代码中用“epoll”指代跨平台事件队列。

### 计算到下一次定时器到期所需的事件（如果有的话）

首先，我们要检测是否有定时器存在。如果存在定时器，那么我们需要计算当前时间与第一个（最先到期的）定时器到期时间的差值。因为我们需要确保我们不会阻塞和遗漏定时器。

```rust, no_run
let next_timeout = self.get_next_timer();

let mut epoll_timeout_lock = self.epoll_timeout.lock().unwrap();
*epoll_timeout_lock = next_timeout;
// We release the lock before we wait in `recv`
// 在`recv`函数中释放这个锁
drop(epoll_timeout_lock);
```
`self.epoll_timeout`是一个`Mutex`，所以我们需要先加锁，才能修改它所持有的值。接下来这点很重要，我们要确保锁`poll`之前被释放。`poll`会挂起我们的线程，尝试在`self.epoll_timeout`时间内拉取事件。

如果我们一直持有这个锁，那么最终会陷入`死锁(deadlock)`的状态。`drop(epoll_timeout_lock)`释放了这个锁。在下一章我们会更详细地介绍这是如何做到的。

### 等待事件就绪

```rust, no_run
if let Ok(event) = self.event_reciever.recv() {
    match event {
        PollEvent::Timeout => (),
        PollEvent::Threadpool((thread_id, callback_id, data)) => {
            self.process_threadpool_events(thread_id, callback_id, data);
        }
        PollEvent::Epoll(event_id) => {
            self.process_epoll_events(event_id);
        }
    }
}
self.run_callbacks();
```

`threadpool`中的线程与`epoll`线程都持有管道`self.event_reciever`的`发送端（sending part）`。线程池中的某个线程完成了一个任务或者`epoll`线程收到了某个事件就绪的提醒，那么就会有一个`PollEvent`被投递到这个管道中，然后在此处被接收。

这个操作会阻塞主线程直到有事件发生，**或者**超时。

> **注意：**
> 
> `epoll`线程会读取`self.epoll_timeout`作为超时时间，所以如果在超时前没有事件发生，那么它会发送一个`PollEvent::Timeout`事件，这么做仅仅是让主循环能够继续，然后处理已经超时的定时器。

根据返回事件的类型是`PollEvent::Timeout`、`PollEvent::Threadpool` 还是 `PollEvent::Epoll`，相应地处理事件。

在之后的章节中我们会解释这些方法。

 ## 5. Check

 ```rust
// ===== CHECK =====
// an set immediate function could be added pretty easily but we won't that here
// 可以把 setImmediate函数添加到此处，但在这里我们并不会这么做。
 ```

接下来，Node实现了一个check的“hook”。调用`setImmidiate`的函数会在这一步被执行。处于完整性考虑，我还是也会将此hook添加进来，但是在本阶段我们不会做任何事情。


## 6. Close Callbacks

```rust
// ===== CLOSE CALLBACKS ======
// Release resources, we won't do that here, it's just another "hook" for our "extensions"
// to use. We release resources in every callback instead.
// 释放资源，但是我们不在此处释放资源。这只是另一个钩子，准备给扩展所使用的。
// 我们在每一个回调中释放资源。
```

这一步，我在注释中已经解释得很多了。通常来说，释放资源在此处完成，比如关闭socket。

## 收尾工作

因为`run`函数基本上就是`Runtime`的开始和结束，所以我们在最后也要进行收尾。接下来的这部分代码是确保所有线程都能结束，释放各自资源和执行所有的析构函数：

首先，遍历`threadpool`的每个线程，发送一个“close”`Task`。接着我们对每一个`JoinHandler`都调用`join`。调用`join`来等待对应的线程执行完毕，这样我们能知道所有的析构函数都执行过了。

接下来，我们会调用`epoll_registrator`的`close_loop()`方法，从而告知操作系统提供的事件队列：我们将关闭事件循环，并释放资源。我们也会`join`epoll线程，直到所有的资源都释放完毕，我们才会结束这个进程。

## 最终的 `run` 函数

```rust, no_run
    pub fn run(mut self, f: impl Fn()) {
        let rt_ptr: *mut Runtime = &mut self;
        unsafe { RUNTIME = rt_ptr };

        // just for us priting out during execution
        // 在执行过程中用来打印循环的轮次
        let mut ticks = 0;

        // First we run our "main" function
        // 首先执行“main”函数（也就是主线程中的函数）
        f();

        // ===== EVENT LOOP =====
        // ===== 事件循环 =====
        while self.pending_events > 0 {
            ticks += 1;
            // NOT PART OF LOOP, JUST FOR US TO SEE WHAT TICK IS EXCECUTING
            // 不是循环的一部分，只是为了让我们更清晰地看到正在执行的时刻
            print(format!("===== TICK {} =====", ticks));

            // ===== 1. TIMERS =====
            // ===== 1. 定时器 =====
            self.process_expired_timers();

            // ===== 2. CALLBACKS =====
            // ===== 2. 回调函数 =====
            // Timer callbacks and if for some reason we have postponed callbacks
            // to run on the next tick. Not possible in our implementation though.
            // 定时器的回调函数，有可能出于某些原因，我们会把这些回调函数放在下一轮循环中执行。
            // 虽然在我们的实现中不可能。
            self.run_callbacks();

            // ===== 3. IDLE/PREPARE =====
            // ===== 3. 空闲和准备 =====
            // we won't use this
            // 我们不会用到这个部分

            // ===== 4. POLL =====
            // ===== 4. 轮询 =====
            // First we need to check if we have any outstanding events at all
            // and if not we're finished. If not we will wait forever.
            // 首先检查是否有尚未解决的事件，如果没有，那么事件循环就结束了
            // 如果有的话，就一直等待
            if self.pending_events == 0 {
                break;
            }

            // We want to get the time to the next timeout (if any) and we
            // set the timeout of our epoll wait to the same as the timeout
            // for the next timer. If there is none, we set it to infinite (None)
            // 计算到下一个定时器的超时的时间差，作为epoll wait的超时时间。
            // 如果没有定时器，那么设置为无限等待。
            let next_timeout = self.get_next_timer();

            let mut epoll_timeout_lock = self.epoll_timeout.lock().unwrap();
            *epoll_timeout_lock = next_timeout;
            // We release the lock before we wait in `recv`
            // 在`recv`函数等待前，释放锁。
            drop(epoll_timeout_lock);

            // We handle one and one event but multiple events could be returned
            // on the same poll. We won't cover that here though but there are
            // several ways of handling this.
            // 虽然我们是一件一件地去处理事件的，但是在一次poll中可能会有多个事件返回。
            // 我们不会在这里讨论这个问题，但是确实有几种方式来处理这个问题。
            if let Ok(event) = self.event_reciever.recv() {
                match event {
                    PollEvent::Timeout => (),
                    PollEvent::Threadpool((thread_id, callback_id, data)) => {
                        self.process_threadpool_events(thread_id, callback_id, data);
                    }
                    PollEvent::Epoll(event_id) => {
                        self.process_epoll_events(event_id);
                    }
                }
            }
            self.run_callbacks();

            // ===== 5. CHECK =====
            // an set immidiate function could be added pretty easily but we
            // won't do that here
            // 可以把 setImmediate函数添加到此处，但在这里我们并不会这么做。

            // ===== 6. CLOSE CALLBACKS ======
            // Release resources, we won't do that here, but this is typically
            // where sockets etc are closed.
            // 释放资源，但是我们不在此处释放资源。
            // 但通常来说，释放资源在此处完成，比如关闭socket
        }

        // We clean up our resources, makes sure all destructors runs.
        // 释放所有资源，确保所有的析构函数都执行。
        for thread in self.thread_pool.into_iter() {
            thread
                .sender
                .send(Task::close())
                .expect("threadpool cleanup");
            thread.handle.join().unwrap();
        }

        self.epoll_registrator.close_loop().unwrap();
        self.epoll_thread.join().unwrap();

        print("FINISHED");
    }
```


## 简化部分

在此，我想提几点显而易见的简化实现，以便你能注意到。在我们的示例中，有许多“例外”没有被包括在内。我们主要关注整体，确保你我之间没有什么误解。`process.nextTick`和`setImmediate` 函数就是这样的两个例子。

我们也不有涉及这样的场景：高负载下服务端可能会有大量的回调函数需要合理地在一次`poll`中执行，这也意味着在等待这些回调执行结束的同时，I/O资源太过空闲。生产环境下的运行时还需要考虑其他类似的情况。

您可能会注意到，实现一个简单的版本对于本书来说已经足够了，但是希望在结束之后，你会发现自己已经能够继续独立、深入地研究下去了。

