# 线程池

我们使用线程池来处理CPU密集型的任务和`epoll, kqueue或者IOCP`无法处理的I/O任务（如对文件系统的操作）。

使用线程池来处理文件I/O的原因很复杂，但是主要是因为文件缓存的方式和硬件驱动的方式，大多数情况下，文件I/O基本上是立即就能`Ready`好的，所以假设在事件队列中等待文件I/O，这实际上没有什么好处，因为大多数操作是几乎立即就能完成的。

第二个原因是尽管Windows平台提供了基于完成情况的模型，但是Linux和macOS却没有。将文件读取到用户进程控制的缓冲区，这可能会需要花费一些时间，而如果我们把文件I/O放置在主循环中进行，可能会阻塞一小会儿主循环，这恰恰是我们想要避免的。

通过线程池来处理这类任务，我们就能确保这些操作不会阻塞主循环，而且当数据在内存中准备完毕时，我们只会被通知一次。

我们需要添加代码来处理来自线程池的事件，非常简短：

```rust, no_run
fn process_threadpool_events(&mut self, thread_id: usize, callback_id: usize, data: Js {
    self.callbacks_to_run.push((callback_id, data));
    self.available_threads.push(thread_id);
}
```

我们接收的参数是`thread_id`、`callback_id`、`data`。这些参数是通过`线程池`中线程共享的channel获取的。

一旦我们获取到这些信息，我们就会把回调函数添加到队列`callbacks_to_run`中，而正如上节所提到的，这些回调会在下次调用`run_callbacks`执行。

最后，我们会用到参数`id`，该参数代表发送已完成任务对应的线程，将其重新放回到可用线程池中。