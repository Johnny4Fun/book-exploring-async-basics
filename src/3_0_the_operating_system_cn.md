# 操作系统

作为程序员，操作系统是我们所做一切工作的中心（除非你正在[编写操作系统](https://os.phil-opp.com/)或者处于[嵌入式领域](https://rust-embedded.github.io/book/)），因此在讨论程序设计中的基础知识之前，我们必须要先进一步深入研究操作系统。

## 操作系统眼中的并发

<div style="color: back;  font-style: italic; font-size: 1.2em">"Operating systems has been "faking" synchronous execution since the 90s."</div>

<div style="color: back;  font-style: italic; font-size: 1.2em">“自90年代以来，操作系统都是在“伪”同步执行。”</div>

这与我在第一章中所说的有关，我说`并发`需要在一个参考系内讨论，我解释了操作系统可能随时停止和恢复你的程序。

在大多数情况下，我们称之为同步代码，实际上是在我们程序员看来是同步的代码。操作系统和CPU都不是存在于完全同步的世界中。

操作系统使用`抢占式多任务处理`机制，只要运行的操作系统是抢占式地调度进程，就不能保证代码按指令一条一条运行而不中断。

操作系统会确保所有重要的进程都能得到CPU的时间片，从而推动进程的执行。

> 对于4/6/8/12个物理内核的现代机器而言，事情并不是那么简单。因为如果系统负载很小，您可能会在一个CPU上不间断地执行代码。这其中很重要的一点是，你不能确定，也不能保证你的代码能不间断地运行。


## 与操作系统打交道

When programming it's often easy to forget **how many moving pieces** that need to
cooperate when we care about efficiency. When you make a web request, you're not
asking the CPU or the network card to do something for you, you're asking the
operating system to talk to the network card for you.

在编程时，当我们关心效率时，常常容易忘记需要合作的模块有多少。当你发出一个web请求时，你不是在请求CPU或网卡为你做某些事情，而是请求操作系统帮你与网卡对话。

作为一个程序员，如果不发挥操作系统的优势，就无法使系统达到最佳效率。因为你基本上无法直接访问硬件。

然而，这也意味着要从头理解所有事情，还需要知道操作系统如何处理这些任务。

为了能够与操作系统一起工作，我们需要知道如何与它打交道（通信），这正是我们接下来要讲的。