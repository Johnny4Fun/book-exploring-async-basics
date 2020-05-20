# 操作系统

原文地址：
[The Operating System](https://cfsamson.github.io/book-exploring-async-basics/3_0_the_operating_system.html)

译者注：

- 已经征得原作者同意，翻译文档和把翻译半成品放在知乎上，详见[issue](https://github.com/cfsamson/book-exploring-async-basics/issues/28)。

- 对于文章中不理解、难翻译的部分，我会咨询作者本人，主要[通过issue交流](https://github.com/cfsamson/book-exploring-async-basics/issues/28)，尽量确保翻译最基本的准确性。大家如果对翻译有异议的、对内容有疑问的都可以提出来，自己在github上[提issue](https://github.com/cfsamson/book-exploring-async-basics/issues/)或者我帮你提。
- 考虑到第一章中英文混合的方法看起来有点杂乱，所以目前采用的翻译方式是把中文翻译独立出来，放在`xxxx_cn.md`，方便对照。[翻译地址](https://github.com/Johnny4Fun/book-exploring-async-basics)



# 正文

作为程序员，操作系统是我们所做一切工作的中心（除非你正在[编写操作系统](https://os.phil-opp.com/)或者处于[嵌入式领域](https://rust-embedded.github.io/book/)），因此在讨论程序设计中的基础知识之前，我们必须要先进一步深入研究操作系统。

## 操作系统眼中的并发

<div style="color: back;  font-style: italic; font-size: 1.2em">"Operating systems has been "faking" synchronous execution since the 90s."</div>

<div style="color: back;  font-style: italic; font-size: 1.2em">“自90年代以来，操作系统都是在“伪”同步执行。”</div>

这与我在第一章中所说的有关，我说`并发`需要在一个参考系内讨论，操作系统随时可能会停止或恢复你的程序。

在大多数情况下，我们称之为同步代码，实际上是在我们程序员看来是同步的代码。操作系统和CPU都不是存在于完全同步的世界中。

操作系统使用`抢占式多任务处理`机制，只要运行的操作系统是抢占式地调度进程，就不能保证代码按指令一条一条地、不间断地运行。

操作系统会确保所有重要的进程都能得到CPU的时间片，从而推动进程的执行。

> 对于动辄拥有4/6/8/12个物理核心的现代计算机而言，事情并不是那么简单。因为如果系统没有什么工作负载，那可能能够在一个CPU上不间断地执行一份代码。这其中很重要的一点是，你不能确定，也不能保证你的代码能不间断地运行。


## 与操作系统打交道

在编程时，我们太关心效率，往往就会容易忘记一个事实：我们需要与很多的部件（CPU、网卡、操作系统等）协同工作。事实上，当你发出一个web请求时，你不是在请求CPU或网卡为你做某些事情，而是请求操作系统替你与网卡进行沟通。

作为一个程序员，如果不发挥操作系统的优势，就无法使系统达到最佳效率。因为你基本上无法直接访问硬件。

然而，这也意味着要从头理解所有事情，还需要知道操作系统如何处理这些任务。

为了能够与操作系统一起工作，我们需要知道如何与它打交道（通信），这正是我们接下来要讲的。