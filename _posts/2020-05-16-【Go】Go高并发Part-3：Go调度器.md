---
layout:         page
title:          【Go】Go高并发Part-2：Go调度器
date:           2020-05-16
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

Go调度器是一个复杂的系统，对我们来说，底层的实现细节并不是最重要的，最重要的是了解调度器是怎么工作的，有着怎么样的行为。了解这些有助于我们充分利用调度器的特性，在开发过程中做最佳决策。

### User Level Threads
上一篇已经说过，内核线程的上下文切换是一件代价高昂的工作。因此才有了用户级线程（User Level Threads）的概念。用户级线程有以下三个特点：
1. 线程完全由运行时系统来管理、调度。这里的运行时系统并非操作系统的一部分，而是用户级的库。
1. 理论上，用户级线程切换快速而且高效，线程切换的代价几乎与一个函数调用相同。
1. 内核对用户级线程无感，内核并不知道有这样的线程存在，也不知道它们如何被调度执行。

### goroutine
goroutine就是Go实现的一种用户级线程，由Go的运行时系统负责管理。一个goroutine占用的资源非常少，这也就表示我们可以创建的goroutine数目比内核线程的数目要多得多。Go runtime scheduler负责调度goroutine运行，这也就是所谓的“原生支持高并发”。在传统多线程程序中，多个线程之间竞争的是CPU，线程需要获取到CPU时间片才能被运行。而在Go程序中，goroutine竞争的是逻辑处理器（Logic Processor），也就是P（后面会有详细介绍）。

### Go Scheduler（模型与算法）
#### Go调度器-MPG模型
Go调度器涉及三个术语：
1. **M** for Machine. M表示逻辑上的Machine，实际就是一个内核线程。M就像是一个勤奋的劳动者，总是在从各种队列中寻找可运行的G来执行。
1. **P** for Processor. P表示逻辑上的Processor，其核心是一些资源，M必须通过这些资源来运行Go代码。P的数量代表了Go执行的并行性，即同一时间有多少个goroutine可以运行。对G来说，P相当于处理器，G只有放入P的可执行G队列中，才能被调度执行。对M来说，P提供了执行G的环境和资源，如任务队列等。
1. **G**. G是Go调度器调度的对象。G存储了goroutine执行的栈信息，goroutine的状态，以及goroutine的任务函数等等，但是G并不执行用户代码。一个G一旦被创建，就不会消失。

MPG模型示意图如下：
![MPG](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/go/scheduler/mpg.png)



#### Go调度器-调度算法
在Go里面，通过限制**P的数量**，可以限制Go程序能够“并行运行”的goroutine数目。这个“并行运行”并不同与内核线程调度中的并行的概念。内核线程的“并行”指的是多个CPU核同时运行多个内核线程，而Go调度器调度goroutine时，多个goroutine只是被Go调度器调度进入多个M中运行。但这些goroutine并不一定能真正地同时运行。实际上，M（内核线程）本身的调度，是操作系统调度器负责的。也就是说，这里的多个M不一定能并行运行。

通过runtime包的GOMAXPROCS函数可以获取以及设置P的数量。GOMAXPROCS会调用stopTheWorld，即暂停所有goroutine的执行。因此，如有需要，应用必须在启动时调用GOMAXPROCS。另外，值得注意的是，对于在集群中部署的docker镜像里面，NumCPU()以及GOMAXPROCS(0)返回的并不一定是真正可用的CPU核数目。





