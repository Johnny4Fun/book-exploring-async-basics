# What's the difference between concurrency and parallelism?并行和并发的区别

Right off the bat, we'll dive into this subject by defining what concurrency is.
Since it is quite easy to confuse "concurrent" with "parallel", we will try to make
a clear distinction between the two from the get-go.

话不多说，让我们通过定义并发来进入今天的主题。因为并发（Concurrency）与并行（Parallel）总是让人傻傻分不清，所以我们首先要给出一个明确的划分。

> Concurrency is about **dealing** with a lot of things at the same time.
>
> 并发是在同一时间段内处理许多任务

> Parallelism is about **doing** a lot of things at the same time.
>
> 并行是在同一**时间点**处理许多任务

We call the concept of progressing multiple tasks at the same time `Multitasking`.
There are several ways to multitask. One is by **progressing** tasks concurrently,
but not at the same time. Another is to progress two tasks at the exact same time in parallel.

我们把同一时间处理多个任务的概念称之为`多任务处理`。多任务处理有很多种方式。第一种是并发处理，不是在同一时间点处理。另一种是在同一时间点，同时处理两个任务。

![parallel vs concurrency](./images/definitions.jpg)

## Lets start off with some definitions一些基本定义

### Resource资源
Something we need to be able to progress a task. Our resources are limited. This
could be CPU time or memory.

支持我们处理任务的条件。资源是有限的，比如CPU时间片、内存等。

### Task任务
A set of operations that requires some kind of resource to progress. A task must
consist of several sub-operations.

需要占用资源来执行的一系列操作。一个任务由多个子操作组成。

### Parallel并行
Something happening independently at the **exact** same time.

同一时间点（同时）独立进行的若干件事。

### Concurrent并发
Tasks that are **`in progress`** at the same time, but not *necessarily* progressing
simultaneously.

若干任务在同一时间段内被处理，但并不意味着这些任务是同时被处理的。

This is an important distinction. If two tasks are running concurrently,
but are not running in parallel, they must be able to stop and resume their progress.
We say that a task is `interruptable` if it allows for this kind of concurrency.

这是非常重要的区别。如果两个任务是并发执行，但是不是并行的，那么这两个任务在进行过程中能被暂停和继续。如果任务支持这种并发操作，则我们称任务是`可中断`的。


## The mental model I use.我的想法

I firmly believe the main reason we find parallel and concurrent programming hard to reason about stems from how we model events in our everyday life. We tend to define these terms loosely so our intuition is often wrong.

我坚信，并行和并发编程难以解释的根源在于我们在日常生活中给事件建模的方式。人们总是倾向于草率地给术语定义，以至于我们的直觉经常出错。

> It doesn't help that **concurrent** is defined in the dictionary as: _operating or occurring at the same time_ which
> doesn't really help us much when trying to describe how it differs from **parallel**
>
> 这是字典中对concurrent的定义：同时处理或发生。然而这对我们区分其与parallel的定义毫无帮助。

For me, this first clicked when I started to understand why we want to make a distinction between parallel and concurrent in the first place!

当我开始理解为什么我们要先区分parallel和concurrent时，这一切就开始有头绪了。

The **why** has everything to do with resource utilization and [efficiency](https://en.wikipedia.org/wiki/Efficiency).

这个原因就是与资源的利用率和效率有关。

> Efficiency is the (often measurable) ability to avoid wasting materials, energy, efforts, money, and time in doing something or in producing a desired result.
>
> 效率是（通常是可测量的）避免在做某事或产生预期结果时浪费材料、精力、努力、金钱和时间的能力。


### Parallelism

Is increasing the resources we use to solve a task. It has nothing to do with _efficiency_.

并行性需要更多完成任务所需的资源，所以它跟_效率_是两码事

### Concurrency

Has everything to do with efficiency and resource utilization. Concurrency can never make _one single task go faster_.
It can only help us utilize our resources better and thereby _finish a set of tasks faster_.

并发与效率和资源的利用率息息相关。然而，并发不能让单个任务处理得更快，它只能帮助我们更恰当地利用资源，从而使得一系列任务能更快地完成。


### Let's draw some parallels to process economics与过程经济学作比较

In businesses that manufacture goods, we often talk about LEAN processes. And this is pretty easy to compare with why programmers care so much about what we can achieve if we handle tasks concurrently.

在生产商品的企业中，人们经常谈论精益流程。这与程序员为什么如此关心并发处理任务所能达到的效果相似。

I'll let let this 3 minute video explain it for me:

<iframe width="560" height="315" src="https://www.youtube.com/embed/Oz8BR5Lflzg" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Ok, so it's not the newest video on the subject, but it explains a lot in 3 minutes. Most importantly the gains we try to achieve when applying LEAN techniques, and most importantly: **eliminate waiting and non-value-adding tasks.**

这并不是精益流程最新的视频，但在三分钟里它阐述了很多。在运用精益流程时，人们想达到的最重要的一点就是：消灭等待和毫无价值的任务。

> In programming we could say that we want to avoid `blocking` and `polling` (in a busy loop).
>
> 在编程中，我们希望避免阻塞和轮询

Now would adding more resources (more workers) help in the video above? Yes, but we use double the resources to produce the same output as one person with an optimal process could do. That's not the best utilization of our resources.

增加资源（增加员工人数）能否有效？当然，但是我们使用双倍资源得到的产出却与一个人采用优化流程所得到的产出相同。显然，资源没有被最大化利用。

> To continue the parallel we started, we could say that we could solve the problem of a freezing UI while waiting for an I/O event to occur
> by spawning a new thread and `poll` in a loop or `block` there instead of our main thread. However, that new
> thread is either consuming resources doing nothing, or worse, using one core to busy loop while checking if
> an event is ready. Either way, it's not optimal, especially if you run a server you want to utilize fully.
>
> 开头提及并发，通过创建新线程，在新线程而非在主渲染线程中进行轮询或阻塞等待，从而及解决因等待I/O事件完成而造成的UI卡住的问题。
>
> 然而，新线程要么是在消耗资源空等，或者更糟一点，占用一个CPU核心一直循环地检查事件是否完成。这两种方式都不是最优的，尤其是当你运行一个需要充分利用资源的服务时。

If you consider the coffee machine as some I/O resource, we would like to start that process, then move on to preparing the next job, or do other work that needs to be done instead of waiting.

如果将咖啡机比作某种I/O资源，那么我们在启动咖啡机后，转而去准备下一项任务，或者做其他一些不需要等待而是需要马上完成的工作。

_But that means there are things happening in parallel here?_

_但这是否就意味着有若干件事情在平行地发生呢？_

Yes, the coffee machine is doing work while the "worker" is doing maintenance and filling water. But this is the crux: _Our reference frame is the worker, not the whole system. The guy making coffee is your code._

确实，咖啡机在运作的同时，员工在维护咖啡机、给咖啡机加水。但这就是关键所在：我们的参考系是员工，而非整个系统。相应地，代码就是那个做咖啡的员工。

> It's the same when you make a database query. After you've sent the query to the database server,
> the CPU on the database server will be working on your request while you wait for a response. In practice, it's a way of parallelizing your work.
>
> 在进行数据库查询时也是如此：当你将查询请求发送给数据库服务后，数据库服务所在的服务器的CPU会处理查询请求，而你在等待请求的回应。实际上，这也是一种将工作平行化的方式。

**Concurrency is about working smarter. Parallelism is a way of throwing more resources at the problem.**

**并发就是要更聪明地工作。并行性是投入更多资源来解决问题的一种方式。**


## Concurrency and its relation to I/O 并发与I/O的关系

As you might understand from what I've written so far, writing async code mostly
makes sense when you need to be smart to make optimal use of your resources.

到目前为止，你大概已经理解了，当你需要优化资源使用时，编写异步代码通常是有效的。

Now, if you write a program that is working hard to solve a problem, there often is no help
in concurrency, this is where parallelism comes in to play since it gives you
a way to throw more resources at the problem if you can split it into parts that
you can work on in parallel.

而如果你在编写的程序是需要大量计算来解决问题的，那么并发通常帮不上什么忙。此时就轮到并行出场了。并行提供了一种给解决问题提供更多资源的方式。如果你可以将问题分解成多个部分，那么就可以并行地处理它了。

**I can see two major use cases for concurrency:**

**我认为的两种主要能够使用并发的情况：**

1. When performing I/O and you need to wait for some external event to occur

    当进行I/O时，你需要等待某些外部事件的发生。

2. When you need to divide your attention and prevent one task from waiting too
  long

  当你需要分散精力，防止任务等待时间过长。

The first is the classic I/O example: you have to wait for a network
call, a database query or something else to happen before you can progress a
task. However, you have many tasks to do so instead of waiting you continue work
elsewhere and either check in regularly to see if the task is ready to progress
or make sure you are notified when that task is ready to progress.

第一种情况的例子就是典型的I/O：在你继续处理任务前，不得不等待如网络调用、数据库请求等操作的完成。然而，有大量的任务要完成，所以你可以选择处理其他的任务，而且要么定时查询是否任务可以继续进行，要么确保当任务能够继续进行下去时，你能够收到通知。

The second is an example that is often the case when having a UI. Let's pretend
you only have one core. How do you prevent the whole UI from becoming unresponsive
while performing other CPU intensive tasks?

第二种情况的例子通常都会有一个UI存在。假设CPU只有1个核心，你要如何避免在执行其他CPU密集型任务时，整个UI界面陷入无响应状态的情况？

Well, you can stop whatever task you're doing every 16ms, and run the "update UI"
task, and then resume whatever you were doing afterwards. This way, you will have
to stop/resume your task 60 times a second, but you will also have a fully responsive UI which has roughly a 60 Hz refresh rate.

每16ms，你可以暂停任何正在进行的任务，然后执行”更新UI“的任务，接着再恢复进行你之前的任务。如此，你在一秒内不得不暂停/恢复任务60次，交换的结果就是，你得到完全响应的UI，其刷新率大约为60hz。

## What about threads provided by the OS?操作系统提供的线程？

We'll cover threads a bit more when we talk about strategies for handling I/O, but I'll mention them here as well. One challenge when using OS threads to understand concurrency
is that they appear to be mapped to cores. That's not necessarily a correct mental model
to use even though most operating systems will try to map one thread to one
core up to the number of threads is equal to the number of cores.

虽然我们之后当我们讨论处理I/O的策略时，会更多地涉及线程相关的内容，但是这里我还是提一下。当使用操作系统提供的线程时，一个难点大概就是线程与CPU核心一一对应。这不一定是一个正确的观念，尽管大多数操作系统会将一个线程映射到一个核心上去，映射的最大数量等于CPU核心数量。

Once we create more threads than there are cores, the OS will switch between our
threads and progress each of them `concurrently` using the scheduler to give each
thread some time to run. And you also have to consider the fact that your program
is not the only one running on the system. Other programs might spawn several threads
as well which means there will be many more threads than there are cores on the CPU.

当我们创建多于CPU核心数的一系列线程时，操作系统通过调度器给每个线程分配运行时间，在这些线程中来回切换，`并发`地处理。同时，要记得你的程序并不是操作系统上唯一在运行的程序。其他的程序也可能会分配若干个线程，这就意味着线程的数量比CPU核心的数量要多得多。

Therefore, threads can be a means to perform tasks in parallel, but they can also
be a means to achieve concurrency.

因此，多线程是并行执行任务的一种方法（多个线程同时处理多个任务），也是实现并发的一种方法（通过线程调度轮流执行多个任务）。

This brings me over to the last part about concurrency. It needs to be defined
in some sort of reference frame.

这也引出了并发的最后一个部分：并发是在某种参考系下被定义的。

## Changing the reference frame改变参考系

When you write code that is perfectly synchronous from your perspective, stop for a second and consider how that looks from the operating system perspective.

当你在编写你自己眼中完全同步的代码时，不妨停一下，思考一下，从操作系统的视角来看，你的代码是怎么样的。

The Operating System might not run your code from start to end at all. It might stop and resume your process many times. The CPU might get interrupted and handle some inputs while you think it's only focused on your task.

操作系统可能不会将你的代码一下子从头到尾执行完，而是可能会暂停/恢复许多次。CPU可能会被中断以处理某些输入，然而你认为它被你的任务所独占。

So synchronous execution is only an illusion. But from the perspective of you as a programmer, it's not, and that is the important takeaway:

所以同步执行只是一个错觉。然而，从你作为一个程序员的角度看，那不是个错觉，相反同步执行是一个非常重要的观点。

**When we talk about concurrency without providing any other context we are using you as a programmer and your code (your process) as the reference frame. If you start pondering about concurrency
without keeping this in the back of your head it will get confusing very fast.**

**当我们单单谈及并发，不涉及任何上下文的时候，我们往往就会把“你”来指代一个程序员，把你的代码（程序）作为参考系（思维定势）。如果你不把这一点抛到脑后，你会经常搞晕的。**

The reason I spend so much time on this is that once you realize that, you'll start to see that some of the things you hear and learn that might seem contradicting really is not. You'll just have to consider the reference frame first.

我花费如此多篇幅在这部分的原因是：一旦你意识到了这一点，你就能理解，一些所见所闻看似矛盾，实则不然，只要先考虑一下参考系。

If this still sounds complicated, I understand. Just sitting and reflecting about concurrency is difficult, but if we try to keep these thoughts in the back of our head when we work with async code I promise it will get less and less confusing.

如果这听起来还是很复杂，我可以理解。单纯思考并发是很困难的，但是当处理异步代码时，如果我们把这些思维定势全抛弃掉，我相信问题会变得明朗许多。