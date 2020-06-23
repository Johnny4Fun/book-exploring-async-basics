# 定时器

## 1. 检查是否有到期的定时器

事件循环的第一步就是检查是否有定时器已经到期，该过程会在`self.check_expried_times()`函数中进行。

```rust
    fn process_expired_timers(&mut self) {
        // Need an intermediate variable to please the borrowchecker
        // 为了满足borrowchecker，所以使用一个中间变量
        let timers_to_remove = &mut self.timers_to_remove;

        self.timers
            .range(..=Instant::now())
            .for_each(|(k, _)| timers_to_remove.push(*k));

        while let Some(key) = self.timers_to_remove.pop() {
            let callback_id = self.timers.remove(&key).unwrap();
            self.callbacks_to_run.push((callback_id, Js::Undefined));
        }
    }

    fn get_next_timer(&self) -> Option<i32> {
        self.timers.iter().nth(0).map(|(&instant, _)| {
            let mut time_to_next_timeout = instant - Instant::now();
            if time_to_next_timeout < Duration::new(0, 0) {
                time_to_next_timeout = Duration::new(0, 0);
            }
            time_to_next_timeout.as_millis() as i32
        })
    }
```

第一个要注意的点：我们需要检查`self.timers`，然后为了理解剩下的语句，我们需要知道`self.timers`是什么类型的集合。

我为这个集合选择的类型是`BTreeMap<Instant, usize>`。理由是我希望这些`Instant`能够按照时间顺序排列。当我们要添加一个定时器时，我会计算出该定时器会在什么时间点到期，再把它添加到该集合中。

> 当希望key有序排列时，BTrees是一种比较好的数据结构。

此处选择`BTreeMap`能让我们通过`range(..=Instant::now())`获取到一个从map的开头直至到NOW这个时间点的区间。

Now I take every key in this range and add it to `timers_to_remove`, and the reason
for this is that I found no good way to both get a range and remove the key's in one
operation without allocating a small buffer every time. You can iterate over the range
but due to the ownership rules you can't remove them at the same time, and we want to
remove the timers, we've run.？？？

然后，我们把区间中的每一个key都添加到`timers_to_remove`，这样做的原因是：我找不到一个好方法能将获取区间和能移除key放在一个操作中，同时又不需要每次都分配一个小的缓冲区。你可以遍历这个区间，但是由于所有权的规则，你不能在遍历的同时又移除它们，而我们恰恰想移除已经执行过的定时器。

事件循环会不停地重复，所以最好要避免在循环中执行分配操作，没必要产生这个开销。

```rust
while let Some(key) = timers_to_remove.pop() {
    let callback_id = self.timers.remove(&key).unwrap();
    self.callbacks_to_run.push((callback_id, Js::Undefined));
}
```

下一步，取出超时的定时器，将其从`self.timers`集合中移除，同时也获取到它们对应的`callback_id`。

As I explained in the previous chapter, this is an unique Id for this callback. What's
important here is that we don't run the callback **immediately**. Node actually registers callbacks to be run on the next `tick`. An exception is the timers since they either have timed out or is a timer with a timeout of `0`. In this case a timer will not wait for the next tick if it has timed out, or in the case if it has a timeout of `0` they will be invoked immediately as you'll see next.

正如先前章节所解释的那样，这里的回调函数都有一个对应的、唯一的id。这里很重要的一点是——我们不会**立即**执行回调。实际上，Node会把这些回调注册到下一轮`tick`执行。除了这些定时器，因为他们要么已经超时了，要么就是超时时间为`0`。在这种情况下，如果定时器已经超时，那就不会等到下一个tick才执行回调，而如果定时器的超时时间为`0`，那么回调就会马上被调用。

不管怎样，我们现在会将回调的id添加到`self.callback_to_run`。

在进入之后的部分前，我们先再复习一下此处使用的`Runtime`结构体：

```rust, no_run
pub struct Runtime {
    pending_events: usize,
    callbacks_to_run: Vec<(usize, Js)>,
    timers: BTreeMap<Instant, usize>,
}
```
