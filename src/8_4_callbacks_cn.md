# 回调

## 2. 处理回调
接下来的步骤是处理我们计划执行的回调。

```rust, no_run
fn run_callbacks(&mut self) {
    while let Some((callback_id, data)) = self.callbacks_to_run.pop() {
        let cb = self.callback_queue.remove(&callback_id).unwrap();
        cb(data);
        self.pending_events -= 1;
    }
}
```

> Shortcut. Not all of Nodes callbacks are processed here. Some callbacks is called
> directly in the `poll` phase we'll introduce below. It's not difficult to implement
> but it adds unnecessary complexity to our example so we schedule all callbacks to be
> run in this step of the process. As long as you know this is an oversimplification
> you're going to be alright :)
>
> 捷径。在Node中，并不是所有的回调都会在此处被执行。我们后面会提到，有些回调会在`poll`阶段直接被调用。这点并不难实现，但是会给我们的示例增加不必要的复杂性，所以我们将所有的回调都安排在这一步骤执行。你只要记得这是一种过度简化的方法就好了:)

在这里，我们`pop`出所有要被安排执行的回调。正如上一章更新的`Runtime`结构体所示，`callbacks_to_run`是一个数组，其中的元素是callback_id和参数类型`Js`组成的元组。

所以当我们获取到一个`callback_id`时，我们就能拿到对应的储存在`self.callback_queue`中的回调，然后删除掉该条目。我们实际获取到的是类型为`Box<dyn FnOnce(Js)>`的回调。之后我们会具体解释该类型，简单来说，这就是一种储存在堆上的闭包，而该闭包接收一个类型为`Js`的参数。

`cb(data)`会执行该闭包中的代码。执行完毕之后，我们就需要将待执行事件的计数器减一：`self.pending_events -= 1;`。

> 这一步非常重要，正如你可能理解的那样，回调中任何长时间运行的代码都会阻塞住`eventloop`，使循环不能继续进行下去。这样既不能处理其他的回调，也不能注册新的事件。这就是阻塞事件循环的代码被排斥的原因。

我们再复习一下本章涉及到的`Runtime`结构体的成员。

```rust, no_run
pub struct Runtime {
    callback_queue: HashMap<usize, Box<dyn FnOnce(Js)>>,
}
```
