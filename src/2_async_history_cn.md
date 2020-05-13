# 异步编程的历史

最初，计算机只有一个CPU（单核），按顺序一条条地执行程序员的指令。没有任务调度、没有线程、没有多任务的概念。这就是很长一段时间里计算机工作的方式。当年所谓的程序就类似打孔纸带。

![Image](./images/punched_card_deck.jpg)

即使在非常早期，还是有很多人在研究操作系统。当80年代个人计算设备开始兴起时，DOS就是当时大多数消费者PC的标配。

这些操作系统通常会将整个CPU的控制权交给当前正在执行的程序，由程序员来完成工作，并为其程序实现任何类型的多任务。这很有效，但是随着使用鼠标的交互式UI和窗口化操作系统成为标准，这个模式就不再管用了。

## 非抢占式多任务处理

第一种方法是通过我们所说的`非抢占式多任务处理`来实现的，它能够保持UI的交互（并运行后台进程）。

这种类型的多任务处理，将操作系统运行其他任务如回应鼠标输入、运行后台任务的职责交到程序员的手中。

通常来说，程序员会将控制权**让给**操作系统。

除此之外，将巨大的责任分摊给每一个在你的平台编写代码的程序员，这自然是容易出错的。程序代码中一个小小的错误可能会引起整个系统的停止或崩溃。

>如果你还记得Windows 95，你应该会记得当一个窗口卡死时，你可以用它画满整个屏幕（就像Windows自带的纸牌游戏的结局一样）。
>
>据说，这是代码中的一个本应将控制权交给操作系统的典型错误。

## 抢占式多任务处理

虽然非抢占多任务处理看起来是个好主意，但是也产生了严重的问题。每个程序与程序员都有控制操作系统UI响应的权限，这可能最终会导致较差的用户体验，因为每一个bug都有可能导致系统停止。

解决这个问题的方式就是将在程序中调度CPU资源的职责交给操作系统。操作系统可以暂停进程的执行，执行其他的操作，然后再切换回去。

在一台单核机器上，你可以想象得到，当运行你写的程序时，操作系统要必须暂停程序来更新鼠标位置，然后再切换回你的程序继续执行。CPU的切换得太频繁了，以至于我们无法分辨出CPU处于繁忙还是空闲状态。

后来，通过切换CPU的上下文使操作系统负责执行任务调度。切换的流程在每秒可以发生许多次，不仅仅是为了保持UI响应，也可以给其他后台任务和I/O事件分配一定的时间片。

这也是现在设计操作系统的主流方法。

> 如果你想要了解更多关于这类多线程的多任务处理，建议你读一读我之前的那被关于[green threads](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/)的书。这是一篇很好的导读书籍，你可以在其中得到你所需要的关于线程、上下文、堆栈和任务调度的基本知识。

## Hyperthreading超线程技术

随着CPU的演进和其越来越多的新增功能 ，比如说若干个算术逻辑单元（ALU）以及额外的逻辑单元，CPU的制造商意识到整颗CPU从没有被完全利用。举个例子，因为一个运算只需要CPU的一部分，所以在同一ALU上可以同时地、并行地执行另一个指令。这就是超线程的开端。

瞧，你的计算机有比如6个核心，12个逻辑核心。这正是超线程技术所带来的效果。这个技术通过使用在线程1执行时CPU的未使用部分来同时执行线程2。实现的方式就是使用一系列聪明的小技巧（就像ALU上的使用的那个一样）。

在超线程技术的加持下，即使我们只有一个单核CPU，事实上我们仍可以将一些工作分配到一个线程上，同时，通过在另一个线程中响应UI事件来保持UI的可交互性，从而更好地利用硬件资源。

> 你可能好奇超线程技术的性能？
>
> It turns out that Hyperthreading has been continuously improved since the '90s.
>**Since you're not actually running two CPU's** there will be some operations that
> need to wait for each other to finish. The performance gain of hyperthreading
> compared to multithreading in a single core seems to be [somewhere close
> to 30 %](https://en.wikipedia.org/wiki/Hyper-threading#Performance_claims) but
> it largely depends on the workload.
> 
> 超线程技术自90年代以来一直在持续地改进。因为你实际上不会同时运行两颗CPU，所以还是会存在一些操作必须要等待另一个完成才能继续进行。在一颗单核处理器上，超线程所带来得多任务处理性能上的提升[在某种某些方面接近30 %](https://en.wikipedia.org/wiki/Hyper-threading#Performance_claims) ，但主要还是取决于工作负载。

## 多核处理器

众所周知，处理器的时钟频率已经很长时间处于一个发展的平稳期。通过改进缓存、分支预测、预测执行、流水线处理等技术，处理器的速度越来越快，然而这些改进的收效开始逐渐减少。

另一方面，新的处理器的越来越小的体积使得我们能够在同样大小的芯片下集成更多的处理器（核心）。如今，大多数的CPU都有多个核心，而大多数的每个核心都支持超线程。

## So how synchronous is the code you write, really?所以你写的代码有多同步，真的吗？

和大多数事情一样，这取决于你看问题的角度。从你的程序和程序代码角度来看，一切通常都会按照你编写的顺序发生。

从操作系统的角度来看，操作系统可能会/也可能不会中断、暂停你的程序，在恢复运行你的程序之前，运行一些其他的程序。

对于CPU而言，它大多数情况下一次只执行一条指令*。CPU不关心谁写的这个程序，所以当硬件中断发生时，CPU会立即停下，将控制权让给中断处理程序。这就是CPU处理并发的方式。


> *然而，现代CPU可以并行地执行很多操作。大多数CPU是支持流水线方式的，这就意味着当前的指令在执行的同时，下一条的指令就会被装载进来。CPU也可能拥有一个分支预测器，用于预测接下来需要载入的指令。
>
> 处理器也可以在不通知程序员和操作系统的前提下，通过使用“乱序执行”来重新排列指令顺序，如果它认为这样能加快处理速度的话。所以不能保证A一定发生在B之前。
>
> CPU将一些工作转移到独立的协处理器上，比如用于进行浮点运算的FPU，将主CPU腾出来执行其他任务等。
>
> As a high-level overview, it's OK to model the CPU as operating in asynchronous
> manner, but lets for now just make a mental note that ****this is a model with some**
> caveats** that become especially important when talking about parallelism,
> synchronization primitives (like mutexes and atomics) and the security of computers
> and operating systems.
>
> 在高级层次上，可以认为CPU是以异步方式运行的，但是一定要记得这只是一个模型，尤其是谈及并行机制、同步原语（如互斥操作mutex和原子操作atomic）以及计算机和操作系统安全时
