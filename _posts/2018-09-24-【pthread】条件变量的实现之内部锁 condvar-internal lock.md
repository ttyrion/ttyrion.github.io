---
layout:     page
title:      【pthread】条件变量的实现之内部锁 condvar-internal lock
subtitle:   分析Glibc(2.27)中条件变量相关的代码
date:       2018-09-24
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

在第一篇《条件变量的实现之原理描述》中介绍过，条件变量的结构体中有一个字段 __g1_orig_size，它会被作为一个futex字来实现一个条件变量内部锁（ **condvar-internal lock**），__g1_orig_size的低2位存储这个内部锁的状态值。__g1_orig_size 的高位bits还有其他作用，因此这个锁会稍微复杂一点。线程调用 pthread_cond_signal() 发送信号时，就要获取和释放这个锁，获取锁和释放锁的函数分别是：__condvar_acquire_lock() 和 __condvar_release_lock()。

这个内部锁有三种状态，是一个“three-state lock”。状态值和对应状态如下(lsb2表示__g1_orig_size/futex字的低2位值)：
1. lsb2 == 0 => not acquired.
2. lsb2 == 1 => acquired.
3. lsb2 == 2 => acquired-with-futex_wake-request.

## 代码分析 condvar-internal lock (three-state lock)

```cpp

//假设lock_value表示__g1_orig_size低2位的值

static void __attribute__ ((unused))
__condvar_acquire_lock (pthread_cond_t *cond, int private)
{
    unsigned int s = atomic_load_relaxed (&cond->__data.__g1_orig_size);
    //lock_value==0, 表示锁没被获取

    while ((s & 3) == 0)
    {
        // RMW，write lock_value=1，成功即表示已获取锁，返回

        if (atomic_compare_exchange_weak_acquire (&cond->__data.__g1_orig_size, &s, s | 1))
        return;
        /* TODO Spinning and back-off.  */
    }

    // 到这里表示，无法把锁状态从0改为1，且__g1_orig_size当前值被load到s中

    // 如果lock_value != 2，尝试把它改为0或者2，然后阻塞在futex上

    while (1)
    {
        while ((s & 3) != 2)
        {
            // lock_value有可能是0，因此if里面还要判断一下

            if (atomic_compare_exchange_weak_acquire
                (&cond->__data.__g1_orig_size, &s, (s & ~(unsigned int) 3) | 2))
            {
                // 如果成功把锁状态变为“acquired”，立即返回，不阻塞线程

                if ((s & 3) == 0)
                return;

                // 如果成功把锁状态变为“acquired-with-futex_wake-request”，则阻塞线程

                break;
            }
        }
        
        //这表示只要fuxtex字(__g1_orig_size)的低2位为2，futex系统调用就会一直阻塞

        futex_wait_simple (&cond->__data.__g1_orig_size, (s & ~(unsigned int) 3) | 2, private);
        
        //被唤醒后重新加载新的__g1_orig_size（的低2位）
        
        //被唤醒的线程，还要继续尝试获取锁，只有成功把锁状态从“not acquired”变为“acquired”的线程才不会阻塞

        s = atomic_load_relaxed (&cond->__data.__g1_orig_size);
    }
}

static void __attribute__ ((unused))
__condvar_release_lock (pthread_cond_t *cond, int private)
{
    //唤醒线程之前，先以原子方式把__g1_orig_size低2位设置为00，并判断之前的值是否为2

    if ((atomic_fetch_and_release (&cond->__data.__g1_orig_size, ~(unsigned int) 3) & 3) == 2)
        //唤醒最多1个线程

        futex_wake (&cond->__data.__g1_orig_size, 1, private);
}

```
分析上面代码，可以发现条件变量把__g1_orig_size的低2位作为一个内部锁来用。假设这个锁初始状态是 “not acquired”，当一个线程成功获得这个锁时，锁状态变为“acquired”，线程不阻塞，继续执行。当第二个线程试图获取这个锁时，会把锁状态改为“acquired-with-futex_wake-request”，该线程阻塞。如果还有第三个线程试图获取锁，它检测到锁当前状态已经是“acquired-with-futex_wake-request”，就立即阻塞。当一个线程释放锁的时候，最多把一个线程唤醒，该线程继续尝试获取锁，只有成功把锁的状态从“not acquired”变为“acquired”的线程才不会阻塞。
