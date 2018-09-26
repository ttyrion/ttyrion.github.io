---
layout:     page
title:      【C++ 11】Memory Order 之 Happens-Before Relation
subtitle:   介绍 Happens-Before 关系
date:       2018-09-23
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 何为 Happens-Before ？
跟内存模型、内存顺序一样，Happens-Before 也不是在 C++ 11 才出现的新概念。

**描述：** Happens-Before 描述的是两个操作访问内存的顺序。假设A、B表示两个有访问内存的操作，如果他们的关系是 A Happens-Before B，那么 A对内存的修改，对另一个线程的操作 B 是可见的。也就是说，如果操作A修改了内存，那么当操作B（可以是在另一个线程）执行时，就能访问到被A修改后的内存。就算这两个线程是在不同的CPU核中运行，甚至两个CPU核有各自独立的高级缓存，拥有 Happens-Before 关系的两个操作A和B也会有上面描述的关系。很明显，Happens-Before 是依赖硬件处理器的。

## 如何获得 Happens-Before 关系
1. **同一线程** 中，如果在代码层面上，一个操作A在另一个操作B前面，那么就有：A Happens-Before B。这是内存顺序的基本规则。
2. **多线程之间** ，可以使用 **acquire/release semantics** 来获得两个操作之间的 Happens-Before 关系。在另一篇《Memory Models and Memory Reordering》中，对 acquire/release semantics 也有过介绍。

## 不要误读
看到 Happens-Before 这个词，大家首先想到的是：A Happens-Before B 的意思是，A操作先发生，B操作后发生，所以A修改的内存对B是可见的。

咋一看，似乎上面的描述没什么问题。可实际上，**根本不是那么回事** ：Happens-Before描述的两个操作之间的关系，根本与时间无关，这是一个描述内存顺序的概念，并不是描述时间上的先后顺序。
1. 如果A、B的关系是Happens-Before关系，并不代表A一定会在B之前执行。比如在同一个线程中，如果B操作没有访问到与A相关的内存，那么即使B操作被重排序到A之前执行，它的结果和在A之后执行是一样的。这种情况下，编译器或者处理器就有可能把B操作重排序到A之前，尽管它们的关系是 A Happens-Before B。这并不违背 Happens-Before 关系，因为B的结果并没有重排序影响。
2. 先后发生的两个操作之间也可能并不是Happens-Before关系。是否Happens-Before关系，要看有没有规则限制从而使A操作对内存的修改，一定是对B操作可见的，而不是看是否操作A先被处理器执行。

**总之，Happens-Before关系与时间无关，只与内存访问有关。**















































































