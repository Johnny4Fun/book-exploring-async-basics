# 清理工作

最后，我们会开始进行收尾工作，join所有的线程，如此一来，我们就知道所有的析构函数都已经执行且没有出错。

```rust, no_run
// We clean up our resources, makes sure all destructors runs.
for thread in self.thread_pool.into_iter() {
    thread.sender.send(Event::close()).expect("threadpool cleanup");
    thread.handle.join().unwrap();
}

self.epoll_registrator.close_loop().unwrap();
self.epoll_thread.join().unwrap();

print("FINISHED");
```

在我们join线程池中的线程之前，我们会发送一个`Close`事件，从而唤醒线程，使线程退出循环。

本书会有专门的章节讨论在`minimio`如何进行收尾工作。

在退出前，join所有的线程，这是一个好习惯，如果你希望知道原因，我推荐[@matklad](https://matklad.github.io/)的这篇文章[Join Your Threads](https://matklad.github.io/2019/08/23/join-your-threads.html)。