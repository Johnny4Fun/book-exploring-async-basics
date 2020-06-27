# I/O事件队列

I/O事件队列用来处理大部分的I/O任务。稍后，我们将讨论如何向队列注册事件，而当事件准备就绪时，它会通过管道来发送`event_id`。

```rust, no_run
fn process_epoll_events(&mut self, event_id: usize) {
    self.callbacks_to_run.push((event_id, Js::Undefined));
    self.epoll_pending_events -= 1;
}
```

稍后我们会看到，我们设计这个队列的方式，实际上，我们将`event_id`和`callback_id`统一为同一类型的值，因为二者都能表示一个唯一的事件。这样做能稍微简化一点代码。

我们将`callback_id`添加到容器中，容器中的元素表示要执行的回调。我们会传入`Js::Undefined`，因为我们没有任何实际的数据需要传递。当我们读到[Http module](./8_3_http_module.md)章节的 时候，你就知道这是为什么了。主要的原因就是I/O队列本身不会返回任何的数据，它只是告诉我们数据已经准备就绪，等待读取而已。

最后，我们需要对记录未完成事件的计数器`epoll_pending_events`做自减，这样我们就能记录还有多少事件未处理完毕。

> **为什么要记录有多少未处理完毕的`epoll_events`？**
>
> 我们不会使用这个值，而我们添加这个值的目的只是为了能够更简单地`打印`出运行时在不同时刻的状态。然而，虽然我们用不到，但记录这些事件的数量还有其他一些好处的。
>
> One area we're taking shortcuts on all the way here is security. If someone were
> to build a public facing server out of this, we need to account for slow networks
> and malicious users.
>
> 之前我们一直没有考虑的一点就是安全。如果有人要基于此构建一个公开的服务，我们需要考虑网络波动和恶意用户。
>
> Since we use `IOCP`, which is a completion based model, we allocate memory for
> a buffer for each `Read` or `Write` event. When we lend this memory to the OS,
> we're in a weird situation. We own it, but we can't touch it. There are several
> ways in which we could register interest in more events than occur, and thereby
> allocating memory that is just held in the buffers. Now if someone wanted to crash
> our server, they could cause this intentionally.
>
> 因为使用了`IOCP`，一种基于完成情况的模型，所以我们需要为每个读/写事件的缓冲区分配空间。当我们把这部分内存提供给操作系统时，我们就会处于一种尴尬的境地。尽管我们有这部分内存的所有权，但我们却不能染指这部分内存。有好几种方式能够注册比发生事件更多的事件的方法有若干种，内存分配发生在缓冲区中。现在，如果有人想破坏我们的服务端，那么他们就会故意制造这样的场景。
>
> 一个比较好的做法是通过记录未完成事件的数量，然后定义一个“最高水位”。当事件数量到达该水位值时，我们就不再向操作系统注册对事件的兴趣，而是将这些事件放在事件队列中。
>
> 进一步说，这也是为什么我们总是应该对这些事件设置超时时间，以便在某一时刻能够回收内存并在必要时返回一个超时error。
