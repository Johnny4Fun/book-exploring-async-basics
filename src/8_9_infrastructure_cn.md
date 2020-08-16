# 基础设施

为了确保一切正常运行，我们现在需要实现一些helper来驱动基础设施的运行。

首先，我们需要定义一个helper，用于获取一个空闲线程对应的`id`。

```rust, no_run
   fn get_available_thread(&mut self) -> usize {
        match self.available_threads.pop() {
            Some(thread_id) => thread_id,
            // We would normally return None and the request and not panic!
            None => panic!("Out of threads."),
        }
    }
```
正如你所见，这里我们大大简化了：如果用完了所有的线程（所有线程都处于繁忙状态），那么我们就会发出`panic!`。这样其实不太好，实际上我们应该实现的逻辑是将所有的请求放入队列中，一旦有线程可用，就执行请求。然而，我们的代码已经足够长了，而且这一点对于我们的目标——理解`async`而言，并不是非常关键。

实现这样一个队列对于读者来说，可能倒还是一个比较好的练习题？你可以随意fork该仓库，然后尝试实现:)

接下来，我们需要为回调函数生成一个唯一的标识符。

```rust

    /// 如果达到最大值，就回到0
    fn generate_identity(&mut self) -> usize {
        self.identity_token = self.identity_token.wrapping_add(1);
        self.identity_token
    }

    fn generate_cb_identity(&mut self) -> usize {
        let ident = self.generate_identity();
        let taken = self.callback_queue.contains_key(&ident);

        // 如果出现冲突或者标识符已经存在的情况，那么我们就会一直循环，直至获取到一个不冲突、不重复的标识符。
        // 我们也不会覆盖如下场景———数量为`usize::Max`的回调等待执行，因为即使速度够快，每纳秒都能在队列中加入一个回调函数，
        // 在64位的系统上要生成这么多数量的回调依旧需要585.5年。
        if !taken {
            ident
        } else {
            loop {
                let possible_ident = self.generate_identity();
                if self.callback_queue.contains_key(&possible_ident) {
                    break possible_ident;
                }
            }
        }
    }
```
`generate_cb_identity`函数实现了上述功能，`generate_identity`则是一个短小精悍的函数，也就是说，我们会尽量避免出现像introduction中出现的那样的超长函数。

>Now, there are some important considerations to be aware of. Even though we use
>several threads, we use a regular `usize` here ？？？and the reason for that is that
>it's only one thread that will be generating Id's. This could cause problems if
>several threads tried to `read` and `generate` new Id's at the same time.
>
>到目前为止，有几个需要注意的关键点。尽管我们使用了几个线程，我们仅仅使用`usize`，原因是同一时间只会有一个线程执行生成id的逻辑。如果有多个线程同时`读取`且`生成`id，那么这样的实现方式可能会出现问题。

我们使用了`usize`的`wrapping_add`方法，通过自增来获取下一个id，这意味着当到达`18446744073709551615` 时，id就会再次回到0。

我们会检测`callback_queue`中是否已经包含该key（尽管在设计层面来看，这是不可能会发生的），而如果该key已经使用，那么就继续生成一个新的key，直至我们发现一个可用的（未被使用过的）key。

接下来，就是将回调添加入`callback_queue`的方法的实现：

```rust
/// Adds a callback to the queue and returns the key

fn add_callback(&mut self, ident: usize, cb: impl FnOnce(Js) + 'static) {
    let boxed_cb = Box::new(cb);
    self.callback_queue.insert(ident, boxed_cb);
}
```
如果之前你没有见过这样的签名`cb: impl impl FnOnce(Js) + 'static`，那么在此我简要解释一下。

`impl ...` 表示我们接收一个参数，该参数实现了`FnOnce(Js)`的trait，并且生命周期是`'static`。而[FnOnce](https://doc.rust-lang.org/std/ops/trait.FnOnce.html)是由`closures`实现的一个trait。在Rust中，`closure`主要可以实现三种trait，而`FnOnce`是当你打算使用消耗掉环境中的一个实例对象时会使用到的一个trait。

由于实现了`FnOnce`的`closure`会消耗/销毁参数，所以该`closure`只能被调用一次。此处的闭包会获取并消耗/销毁之前我们在`main`线程中创建的资源。因为我们希望资源一旦被使用，就需要利用Rust里的`RAII`方式进行回收。在本用例中，`FnOnce`会隐式地返回`()`，所以我们不需要写成`FnOnce(Js) -> ()`.

在此处，因为回调函数注定只会被调用一次，这恰恰完美发挥了`FnOnce`的作用。

traits没有所谓的size，所以为了使编译器能够在栈上为其分配空间，我们要么就是获取一个引用`&FnOnce(Js)`，要么就使用`Box`将其分配在堆上。我们采取的策略是后者，因为对于我们的示例，这是唯一能行得通的做法。Box是一个指针，指向分配在堆上的变量，因为已知占用内存大小，所以我们将引用储存在名为`callback_queue`的HashMap中。

>什么是闭包？
>
>在rust中，一个函数的定义可以很简单，形如`||{}`。这可以等同于一个函数指针，也等同于指向`my_method`（没有圆括号）。只要通过引用环境变量来将不属于`function`拥有的变量“包”住，就形成了闭包。
>
>`Fn`traits是自动被实现的，而闭包是实现`Fn`、`FnMut`还是`FnOnce`，取决于闭包是转移了一个无法拷贝的变量的所有权、取走了一个共享引用`&`还是一个独占引用`&mut`（通常被称为可变引用）。

现在我们已经掌握了一些闭包基本知识，那么可以开始下一步了。接下来要实现的方法是用来注册`I/O`相关的工作，即我们向运行时注册一个`epoll`事件的方法。

```rust
pub fn register_event_epoll(&mut self, token: usize, cb: impl FnOnce(Js) + 'static) {
    self.add_callback(token, cb);

    print(format!("Event with id: {} registered.", token));
    self.pending_events += 1;
    self.epoll_pending_events += 1;
}
```

首先，我们通过调用之前提到的方法，将回调添加到`callback_queue`中。接着，我们执行一个打印语句，因为我们想展示程序的流程，所以我们需要在合适的关键位置添加打印语句。

> 需要注意的一个关键点：在这里的`token`已经确定是唯一的，通过`Http`模块（在我们的示例中唯一会使用该方法的注册事件）生成的。至于为什么会使用到该模块，我们会在之后的一个小章节中解释。现在只需要注意的是，我们并不会在此处使用`generate_cb_identity`。

我们会为`pending_events`和`epoll_pending_events`对应的计数器分别+1。

下一个方法的用途是向线程池投递任务。

```rust
pub fn register_event_threadpool(
    &mut self,
    task: impl Fn() -> Js + Send + 'static,
    kind: ThreadPoolTaskKind,
    cb: impl FnOnce(Js) + 'static,
) {
    let callback_id = self.generate_cb_identity();
    self.add_callback(callback_id, cb);

    let event = Task {
        task: Box::new(task),
        callback_id,
        kind,
    };

    // we are not going to implement a real scheduler here, just a LIFO queue
    // 我们并不会在此实现一个真正的调度器，而是简化为一个LIFO队列
    let available = self.get_available_thread();
    self.thread_pool[available].sender.send(event).expect("register work");
    self.pending_events += 1;
}
```
先看一下该函数的参数（`&mut self`除外）。

`task: impl Fn() -> Js + Send + 'static` 表示一个我们想要在一个线程上执行的任务。这个闭包的约束是：`Fn() -> Js + Send + 'static`，表示该闭包不接收任何参数，而且返回一个`Js`类型的对象。它必须是`'Send`的，因为我们会把该任务投递给其他线程执行。

`kind: ThreadPoolTaskKind` 用于指示任务的类型。这样做的原因有两个：

1. 我们需要能够向线程发送`Close`事件。
2. 我们希望能够打印出收到的每个任务的具体类型。

正如你所理解的那样，并不一定要为每个task都创建一个`Kind`，但是因为我们希望能够打印出线程收到的东西，所以我们需要某种方式来判断每个线程收到的具体任务类型。

`cb: impl FnOnce(Js) + 'static` 最后一个参数是回调函数。`task`返回一个`Js`类型对象，而回调函数恰好接收`Js`作为参数，这二者并不是一个巧合。因为此处在线程中执行的task的返回值，即为回调函数的输入。而此处的闭包并不需要是`Send`，因为我们并不会直接将回调本身传入线程池。

接下来，我们通过`self.generate_cb_indentity()`生成一个新的id，然后将回调添加到回调队列中。

接着，我们会创建一个新的`Event`，而且正如之前我所操作的那样，这里我们也需要使用`Box`来装载闭包。

现在最后一部分可能会相对复杂一些。在这部分中，你需要决定如何调度线程池中的任务。在我们的示例中，我们每次只会获取一个空闲线程（而且如果线程用完了，那就糟了——`panic!`），然后将任务投递给该线程，从而执行该任务，直至任务完成。

你可以依据任务类型`TaskKind`来分配优先级，确定哪些任务是简单的，哪些是复杂的，而后根据当前的负载来优先处理哪些任务。这部分能做的工作很多。虽然我们会选择最简单的调度方式——直接将任务依照它们发生的顺序投递给一个空闲线程。

“基础设施”的最后一个部分是一个函数，用来设置定时器。

```rust
    fn set_timeout(&mut self, ms: u64, cb: impl Fn(Js) + 'static) {
        // Is it theoretically possible to get two equal instants? If so we'll have a bug...
        // 理论上来说，有没有可能出现获取到两个相同时刻？如果是这样的话，那这就会成为一个bug...
        let now = Instant::now();
        let cb_id = self.generate_cb_identity();
        self.add_callback(cb_id, cb);
        let timeout = now + Duration::from_millis(ms);
        self.timers.insert(timeout, cb_id);
        self.pending_events += 1;
        print(format!("Registered timer event id: {}", cb_id));
    }
```

设置定时器时，需要调用`std::time::Instant`获取一个代表“now”的对象。 这是我们要做的第一件事情，因为该函数的使用者希望超时时间是从“now”开始进行计算的，而此处其他的一些操作可能会消耗一些时间。

我们会为传入`set_timeout`的回调`cb`生成一个标识符，然后将回调加入到回调队列中。

我们给`Instant`加上`duration`（以毫秒为单位），这样我们就能知道定时器何时会超时。

我们将计算出来的超时时刻`Instant`作为key，`callback_id`作为value，插入到`BtreeMap`中。

接着我们会将`pending_vents`对应的计数器+1，并且打印一条消息，以便能够跟踪程序的流程。

> 这可能是一个比较合适的时间点来简单谈谈为什么我们选择`BTreeMap`作为存储定时器的容器。
>
> 在文档中我们可以看到”理论上来说，二叉搜索树（BST）是一个排序的map的最优底层实现，因为一个完全平衡二叉树在理论上能使用最少的比较次数来查找到一个元素(log2n).“
>
> 这里使用的不是二叉树，而是B树。BST为每个值分配一个节点，而BTree为每个节点分配一个小的Vec用于存放每个节点的值。现代计算机向缓存中读取的数据比我们通常要求的要多得多，这也是它们喜欢内存中连续部分的原因之一。一个BTree会产生一个更优的“缓存效率”，收益理论上通常会优于使用一个真正的BST算法。
>
> 最后，既然我们在这里讨论的是有序集合的搜索，而且定时器就是一个很好的例子，而且Rust的标准库中也有可用的实现，所以我们当然会使用BTreeMap咯。
