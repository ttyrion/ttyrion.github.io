---
layout:     page
title:      【C++ 11】Memory Models and Memory Reordering.md
subtitle:   C++ 11 引入了多线程、并发等等概念，伴随着的是新的内存模型。
date:       2018-09-23
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 前言
Memory Models 和 Memory Reordering 并不是由 C++ 11 提出的，它由来已久，
只是因为 C++ 11 中引入了多线程和并发，所以就同时引入了在多线程编程领域存在已久的概念：Memory Models 和 Memory Reordering。
在单线程中，Memory Reordering（包括编译器的reordering 和 处理器的reordering） 一样存在，只不过单线程的情况下，我们不会感知到它的影响。
而在多线程中，情况就不同了。

多线程程序中，经常需要在线程之间共享数据，这里提到的“共享”，包括了reads和writes（如果多个线程都是reads操作，那我们也不会感知到 Memory Reodering 的影响）。
如果多个线程同时读写共享的内存，就会导致内存错误。如果 reads／writes 操作是非原子的，这种情况是最糟糕的，因为一个线程读到的内存中的数据是不可预知的，它很有可能读到“不完整的”数据，因而使整个程序出现异常行为。

为了防止程序出现上述错误，就需要一些技术来实现多线程同步访问共享的数据。  
最简单的方式就是使用锁（lock）。然而，线程获取锁（acquire a lock）和释放锁（release a lock），都带来性能问题。如果 acquire／release 操作很频繁，情况尤为糟糕，性能会显著降低。

另一种技术叫做无锁编程（**Lockless programming**），这种技术是为了在不加锁的情况下，在多线程之间安全地访问共享数据。进行无锁编程时，有两个问题必须要处理：
1. 非原子操作；
2. 重排序。

## 非原子操作 （Non-Atomic Operations）  
一个**原子操作**，指的是一个不可分割的操作：其他线程不可能看到这个操作只完成一部分的情况。原子操作对 lockless programming 非常重要，因为没有原子操作的话，其他线程可能读到还没写完的值（half-written values），或者访问到其他不一致的状态。

对于现代处理器，我们可以假设对于自然 **对齐** 的基本数据类型的读写是原子的，比如对与 int 和 char 等等。只要该类型不比内存总线更宽，CPU 就可以在一次总线交换时完成读写操作，这使得其他线程不可能访问得到只读写一半的内存。
请注意：就算一个类型与内存总线一样宽或者更小，如果内存不对齐，也是没法在一次总线交换中就能读写完成的。
**另外，C++ 标准并没有明确要求基本数据类型（比如 int 和 char 等）的读写是原子的，即便是C++ 11, 也只是要求std::atomic<int>、std::atomic<char>才是的读写是原子的。**

我们可以通过下面的方式来保证操作的原子性：
1. 自然的原子操作
2. 用锁来封装组合操作（当然，Lockless 编程中我们不用锁）
3. 操作系统提供的组合操作的原子版本函数，比如 Windows 的 InterlockedExchange。

## 重排序（Reordering）
重排序是一个更加复杂晦涩的问题。Reads／Writes 操作不总是按照我们的源代码的顺序来执行的，这可能导致很令人不解的问题。而，**编译器** 以及 **处理器** 的会做写重排序的工作。
在多线程编程中，一个线程写入一些数据，然后设置了一个flag变量，通知其他线程数据已经准备好。这称之为 **write-release**。如果这两个写操作被重排序，其他线程就可能在数据被那个线程写入内存之前读取到了一个被设置了的flag（即flag的写操作先于数据的写操作发生了）。
类似的，一个线程读一个flag，如果flag表明另一个线程已经准备好了数据的话（即当前线程获取到了数据访问权），就读出共享的内存数据。这称之为 **read-acquire**。如果这两个读操作被重排序的话，那这个线程可能就读到了旧的数据（非最新的数据）。实际上，这个线程很可能读到的是只写了一半的数据（乱的数据）。

## 内存模型 （Memory Models）
内存重排序有很多种，处理器和我们使用的开发工具链，决定最终发生什么样的内存重排序。
一个**内存模型**可以告诉我们，对于给定的处理器和工具链，我们可以预期会发生什么样的内存重排序（相对与我们源代码中的顺序）。
**再次说明，内存重排序只有在使用 lock-free programming 的多线程编程中才会对我们有影响。**

内存模型：  
1. Really Weak
2. Weak with data dependency ordering
3. Usually strong(implicit acquire／release)
4. Sequentially Consistent（比如，不开启优化的单核情况）
上面的四种内存模型从上往下，下面的内存模型保证了上面的模型的重排序行为，并提供更多的保证。

内存模型可以分为两部分来看： **软件（编程语言）内存模型** 和 **硬件内存模型**。  
1. 硬件内存模型告诉我们，处理器执行指令时，会发生什么样的相对于机器代码（汇编代码）的内存重排序。
2. 软件内存模型，这也是通常我们唯一需要考虑的内存模型，比如 C++ 11， Java 的内存模型。

### Weak Memory Models
这是最弱的（内存顺序限制最少的）内存模型，任何读写操作可能被其他读写操作重排序，只要不影响一个独立的单线程的行为。实际上，此时内存的重排序可能是由于编译器，也可能是处理器产生的。
weak memory model，也可以说是 **relaxed memory model**。C++ 11 的新内存模型中，就包括了relaxed。以 C++ 11 为例，我们使用底层的原子操作的时候，不需要关心处理器是什么样的内存模型。
我们只需正确地设置编程语言层面上的内存排序限制，来限制编译器重排序。

### Weak with Data Dependency Ordering
有些处理器的内存模型与上面的Weak内存模型基本一致，只是提供了额外的特性：这些处理器维护 data dependency ordering。
什么是data dependency ordering？以C++为例，如果我们写这样的代码：B = A; 那么我们可以确定我们读到的B的值，至少和A一样新。

### Strong Memory Models
在strong memory model中，每条机器指令的执行顺序都隐式符合 **acquire／release** 语义。其结果是：当一个CPU核执行了一系列的写操作时，其他所有的CPU核可以看到的那个内存数据的更新顺序就是它被写的顺序。
**acquire／release semantics** ：其实这个概念并没有标准定义，以下的定义来自MSDN的一篇文章。
1. **Acquire semantics** 仅能被应用到读取共享内存的操作中，不论是RMW（Read-Modify-Write）操作或简单的load操作。正如上面已经提到过的，这个操作就被认为是一个 **read-acquire**。Acquire semantics阻止源代码中read-acquire操作后面的reads/writes操作被重排序（到read-acquire前面）。
2. **Release semantics** 仅能被应用到写入共享内存的操作中，不论是RMW操作或简单的store。正如上面也提到过，这个操作就被认为是一个 **write-release**。Release semantics 阻止源代码中write-release操作前面的任何reads/writes操作被重排序（到write-release后面）。

理解了上述的两种语义的定义后，就能发现，简单地组合使用内存屏障（memory barrier）类型，就能获得acquire和release语义。memory barriers必须被放到write-release操作的前面，read-acquire操作的后面。
一种获得memory barrier的方式，是显示调用硬件平台提供的 **屏障指令**（fence instructions，我译为屏障指令）。  
除此之外，还能使用语言提供的方式。以 C++ 11 为例。C++ 11 的atomic标准看定义了 std::atomic_thread_fence(std::memory_order order)，它的参数指定fence类型，可取值如下：
```cpp
typedef enum memory_order {

    memory_order_relaxed,
    memory_order_consume,
    memory_order_acquire,
    memory_order_release,
    memory_order_acq_rel,
    memory_order_seq_cst
} memory_order;

```

### Sequential Consistency
这是限制最强的一种内存模型，在这种内存模型中，不会发生内存重排序。在现代处理器中，很难发现一个多核处理器在硬件层面上保证Sequential Consistency内存模型。通常，这种内存模型只在软件内存模型中存在（即由编程语言提供），比如上面 C++ 11 的 std::memory_order_seq_cst。
