---
layout:     page
title:      【C++ 11】Memory Order 之  Synchronizes-With Relation
subtitle:   介绍 Synchronizes-With 关系
date:       2018-09-23
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 何为 Synchronizes-With ？
Synchronizes-With 术语是由编程语言设计者用来描述 **源代码** 层面上的操作（**可以是非原子操作**）怎么影响内存的。编程语言利用 Synchronizes-With 来保证一个线程对内存的修改，对其他线程是可见的。

在 Synchronizes-With 关系中，对共享内存的修改不需要是原子的。实际开发中，共享内存经常是不能原子操作的，我们只需控制一个线程完成内存修改之后，再允许其他线程访问这块共享内存。

Synchronizes-With 也不是 C++ 11 提出的新概念，其他编程语言，例如 Java，也有这个概念。在所有的语言对 Synchronizes-With 的规范说明中，有一个共同点：如果两个操作（通常是两个不同线程中的操作）之间是 Synchronizes-With 关系，则这两个操作同时有 **Happens-before 关系**。

C++ 11 标准并没有过多介绍 Synchronizes-With，标准只需制定 Synchronizes-With 的规范。以下基本就是 C++ 11 标准 对这个术语的描述了：
```
An atomic operation A that performs a release operation on an atomic object M synchronizes with an atomic
operation B that performs an acquire operation on M and takes its value from any side effect in the release
sequence headed by A.

```
上面那段话，看似简单，实则可以由此推导出一个 Synchronizes-With 的“编程模型”。

## Synchronizes-With 的“编程模型”: 2个要素
仔细分析上面那段话，可以总结出一个 Synchronizes-With 关系中的 **两个关键要素**：
1. atomic object M，对它的操作是保证 Synchronizes-With 关系的关键，可以称之为 **guard object**。
2. 我们通过操作 guard object 使得 A 和 B 之间获得 Synchronizes-With 关系的目的，是为了共享内存，可以称之为 **payload**。

## 示例：A write-release can Synchronize-With a read-acquire
假设我们的共享结构体Task定义如下：
```cpp
struct Task {
    char task_msg[128];
    int  task_id;
};

std::atomic<int> my_guard(0); 
Task my_shared_task;

```
my_shared_task（Synchronizes-With 关系的两个要素之一，**payload**）就是我们打算被多个线程共享的对象。通常，向内存中的这样一个对象的写操作，显然不是原子的（它的宽度超过内存总线）。为了在多个线程直接安全地共享my_shared_task，需要用到 acquire/release 语义（在我另一篇《Memory Models and Memory Reordering》中有介绍），这就会用到 Synchronizes-With 关系的第二个要素，**guard**。  
假设我们有一个producer线程执行以下任务：
```cpp
void produce(void* param) {
    // write shared memory 
    
    //注意，这个不是原子操作，Synchronizes-With 不要求payload的读写是原子的

    std::fill(std::begin(task.task_msg), std::end(task.task_msg), 1);

    // atomic write-release
    my_guard.store(1, std::memory_order_release);
}

```
假设另一个consumer线程执行以下任务：
```cpp
void consume(void* param) {
    // atomic read-acquire

    int data_ready = my_guard.load(std::memory_order_acquire);
    if (data_ready != 0) {
        // Synchronizes-With 保证了此时我们读到的 my_shared_task 是最新的，并且是完整的

        handle_task(my_shared_task);
    }
}

void handle_task(const Task& task) {
    //...
}

```
**std::memory_order_release** 和 **std::memory_order_acquire** 做了什么工作？
1. 在write-release编程中，std::memory_order_release要求编译器（可能还有处理器）保证当前操作（这里的my_guard.store()）之前的读写，不会被重排序到该操作之后。
2. 在read_acquire编程中，std::memory_order_acquire要求编译器（可能还有处理器）保证当前操作（这里的my_guard.load()）之后的读写，不会被重排序到该操作之前。
两者结合，就能保证上面的consume中读取到的my_shared_task是被produce()更新后的数据（up to date）。

再结合这个例子和标准的描述，不难发现例子中的 my_guard 就是标准描述中的“atomic object M”，而“atomic operation A”对应例子中的my_guard.store()， “atomic operation B”对应例子中的my_guard.load()。

**注意，**标准的描述中，还对操作B有一个条件限制：“takes its value from any side effect in the release
sequence headed by A”。也就是说，操作B还要读到被操作A（以release内存顺序）修改过的“atomic object M”的值。为什么有这么一句话呢？看下面。

## Synchronizes-With 不是源代码层面上的顺序关系
以上面的示例来说，如果consumer线程的my_guard.load(std::memory_order_acquire)先被执行，而此时producer线程的my_guard.store(1, std::memory_order_release)还未执行，那么就不能说my_guard.store(1, std::memory_order_release) 和 my_guard.load(std::memory_order_acquire)是 Synchronizes-With 的关系。显然，这种情况下，consumer线程读不到producer线程更新的共享内存数据。也就是说，两个操作是否是 Synchronizes-With 的关系，还要看 load 读到的 **guard object** 的值。这是标准描述中给“atomic operation B”的第二个限制条件（第一个是 “performs an acquire operation”）。

因此，Synchronizes-With 是运行时的一种关系，而不是源码中的代码语句之间的关系。

## 如何获得 Synchronizes-With 关系？
一组 write-release/read-acquire 操作，并不是获得 Synchronizes-With 关系的唯一方式。C++11 atomics 也不是获得 acquire/release 语义的唯一方式。

除了上面示例中的lock-free编程方式，C++ 11(还有其他语言，如Java)的锁机制，也能提供Synchronizes-With。举例来说，释放一个mutex，和随后的获取一个mutex，两个操作之间也是Synchronizes-With关系。在这里，mutex就是一个guard object，负责提供安全的方式让多线程共享payload。

下面附上一张图，来自我参考的一篇文章，原文链接在最后。
![synchronize](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/sc/synchronizes-with.png)

### 参考
[The Synchronizes-With Relation](http://preshing.com/20130823/the-synchronizes-with-relation/)

