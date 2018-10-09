---
layout:         page
title:            【pthread】条件变量的实现之等待和发送信号
subtitle:     分析Glibc(2.27)中条件变量相关的代码
date:             2018-09-24
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

在第一篇《条件变量的实现之原理描述》中介绍过，Glibc实现条件变量时，把等待线程（稍后都称为waiters）分为两组：G1和G2。一个线程（waiter）调用pthread_cond_wait()成功阻塞时，就会被放入G2组中：

```cpp

// __pthread_cond_wait_common(...)

uint64_t wseq = __condvar_fetch_add_wseq_acquire (cond, 2);

// Get the index of G2.

// A waiter always go into G2 after acquiring its position.

unsigned int g = wseq & 1;
uint64_t seq = wseq >> 1;

...

if (abstime == NULL)
{
    // Block without a timeout.
    
    err = futex_wait_cancelable(cond->__data.__g_signals + g, 0, private);
}

...

```

何为“把等待线程放入G1或者G2”?，实际上就是让等待线程阻塞在__g_signals[g1]或者__g_signals[g2]这个futex字上。

看上面代码，可以发现 pthread_cond_wait “打算”把调用线程都加入了G2这一组等待队列中。为什么说“打算”呢？因为实际上，在 pthread_cond_wait 把调用线程阻塞在__g_signals[g]前，可能发生了 G1 和 G2 的角色切换。

### pthread_cond_signal 可能唤醒一个以上的等待线程
```cpp
// __pthread_cond_wait_common(...)

...
do
{
    while (1)
    {
        unsigned int spin = maxspin;
        while (signals == 0 && spin > 0)
        {
            // Check that we are not spinning on a group that's already closed.
            
            if (seq < (__condvar_load_g1_start_relaxed (cond) >> 1))
                goto done;

            signals = atomic_load_acquire (cond->__data.__g_signals + g);
            spin--;
        }

        // If our group will be closed as indicated by the flag on signals, don't bother grabbing a signal.
        
        if (signals & 1)
            goto done;

        // If there is an available signal, don't block.
        
        if (signals != 0)
            break;

        ...
    }
}

```

```cpp
// __pthread_cond_signal(...)

if ((cond->__data.__g_size[g1] != 0)
  || __condvar_quiesce_and_switch_g1 (cond, wseq, &g1, private))
{
  atomic_fetch_add_relaxed (cond->__data.__g_signals + g1, 2);
  cond->__data.__g_size[g1]--;
  do_futex_wake = true;
}

__condvar_release_lock (cond, private);

//wake up 1 thread

if (do_futex_wake)
    futex_wake(cond->__data.__g_signals + g1, 1, private);

```
看上面的代码，一个线程A（**waiter**）调用pthread_cond_wait()，假设此时g1=1，g2=0。还未准备阻塞前，线程A会先自旋一段时间，检测当前组G2中是否有可用信号。而另一个线程B（**signaler**）在此期间调用了pthread_cond_signal()，线程B判断到当前__g_size[g1]==0（即__g_size[1]==0），就会执行__condvar_quiesce_and_switch_g1()，之后g1=0，g2=1。接着线程B会给__g_signals[g1]中添加一个条件信号，并且把do_futex_wake设置为true。线程A本来在自旋中检测G2是否有信号可用的，由于线程B切换了G1和G2，使得线程A检测到了来自新的G1的可用信号并且跳出循环，不再阻塞。而线程B设置了do_futex_wake=true后，又调用了一次futex_wake()，这很可能又唤醒一个线程。

在上面的场景中，调用一次pthread_cond_signal()，唤醒了两个等待线程。


### __g1_start 的设置
__g1_start 的设置比较晦涩，却很重要。因为正是它使得“伪唤醒”得以实现，这里也会说明为什么一定要实现“伪唤醒”。

```cpp
// __condvar_quiesce_and_switch_g1(...)

// Relaxed MO is fine because the change comes with no additional constraints

// that others would have to observe.

__condvar_add_g1_start_relaxed (cond, (old_orig_size << 1) + (g1 == 1 ? 1 : - 1));

...

// These values are just observed by signalers, and thus protected by the lock.
  
unsigned int orig_size = wseq - (old_g1_start + old_orig_size);
__condvar_set_orig_size (cond, orig_size);

// Use and addition to not loose track of cancellations in what was previously G2.

cond->__data.__g_size[g1] += orig_size;
  

```
首先，我们可以知道，__g1_start 的值是不断增加的，其次，它的值和之前的 __g1_orig_size 值有关。

(old_orig_size << 1) + (g1 == 1 ? 1 : - 1)，这个表达式咋一看还以为 __g1_start 的值是 __g1_orig_size 的值乘以2再加上1或者-1。实际上，因为 __g1_start 的 LSB 即为当前G2的索引，所以实际上G1的起始位置不能包含LSB。因此这个 __condvar_add_g1_start_relaxed 只是做了两件事情：
1. 设置 G1 的起始位置为 old_orig_size（之前的__g1_orig_size 值）。
2. 设置 __g1_start 的最低位为新的 G2 的索引（也就是老的 G1 的索引）。

那么实际上可能是这样的：一开始，__g1_orig_size 和 __g1_start 的值都是0。当有N个线程阻塞在 G2 组后，一个signaler线程使得 G1 和 G2 切换（调用__condvar_quiesce_and_switch_g1()），会设置相关的几个值（假设 g1_start 表示 __g1_start 中表示G1起始位置的部分）：
```
wseq = N;
g1_start = 0;
__g1_orig_size = N;
__g_size[g1] = N;

```

假设第二次有M个线程阻塞在 G2 组后发生 G1 和 G2 切换，这几个值如下：
```
wseq = N + M;
g1_start = N;
__g1_orig_size = M;
__g_size[g1] = M;

```
也就是说：等待序列 **__wseq** 是递增的（这也就是为什么还要增加一个字段 **__wrefs** 来作为引用计数），G1组的起始位置也是递增的。












