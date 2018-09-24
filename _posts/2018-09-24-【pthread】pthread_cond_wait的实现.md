---
layout:     page
title:      【pthread】pthread_cond_wait 的实现
subtitle:   分析Glibc(2.27)中pthread_cond_wait源代码
date:       2018-09-24
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 前言
这里关于 pthread_cond_wait 的讨论，同样适用于 pthread_cond_timedwait。先看看这两个函数的实现：

```cpp
/* See __pthread_cond_wait_common.  */
int __pthread_cond_wait (pthread_cond_t *cond, pthread_mutex_t *mutex)
{
  return __pthread_cond_wait_common (cond, mutex, NULL);
}

/* See __pthread_cond_wait_common.  */
int __pthread_cond_timedwait (pthread_cond_t *cond, pthread_mutex_t *mutex,
    const struct timespec *abstime)
{
  /* Check parameter validity.  This should also tell the compiler that
     it can assume that abstime is not NULL.  */
  if (abstime->tv_nsec < 0 || abstime->tv_nsec >= 1000000000)
    return EINVAL;
  return __pthread_cond_wait_common (cond, mutex, abstime);
}
```
从上面的代码来看，pthread_cond_wait 和 pthread_cond_timedwait 的实现是一模一样的，最终都是调用同一个函数 __pthread_cond_wait_common 来实现。因此，我们只需要分析 __pthread_cond_wait_common。简洁起见，后面就只说 pthread_cond_wait，不再提 pthread_cond_timedwait。

## POSIX 描述
首先来看看，POSIX标准文档对 pthread_cond_wait 的描述，我列出了主要的几点：
1. pthread_cond_wait 应该阻塞在一个条件变量condvar上。应用程序应该保证调用这两个函数时传的参数 mutex 已经被调用线程获取（locked／acquired），否则会报错（对于PTHREAD_MUTEX_ERRORCHECK和robust的mutexes），或者结果可能是 **undefined behavior**（对于其他mutexes）。
2. pthread_cond_wait 会执行一个 **原子操作**（atomical operation）: 释放mutex，并且阻塞调用线程（release the mutex and block the calling thread on the condvar）。因为这个操作是原子的，所以当这个mutex被释放以后，如果有另一个线程获取到了这个mutex，并且调用了pthread_cond_broadcast()或者pthread_cond_signal()，这对前面那个释放mutex并阻塞的线程也会起作用。
3. 使用条件变量时，通常会同时配合使用一个Boolean类型的断言变量flag，如果为true，表示这个线程可以继续执行。然而，pthread_cond_wait() 可能会被**“伪”唤醒**。**也就是说，pthread_cond_wait()返回，并不表示flag为true或者为false。必须重新取flag的值来判断。**
4. 当一个线程用一个mutex调用 pthread_cond_wait()阻塞时（在等待条件变量condvar），这个 mutex 和 condvar 就被动态绑定起来，且这种绑定关系会一直存在，只要还有线程阻塞在这个condvar上。在此期间，如果有一个线程尝试用另外一个mutex来调用pthread_cond_wait()期待等待同一个condvar，那么就会导致 **undefined behavior**。当所有等待线程被唤醒（比如被一个pthread_cond_broadcast()），下一个使用mutex（可能跟前面那个不同）调用pthread_cond_wait()来等待那个condvar的操作，就会再次动态绑定这个mutex和condvar。也就是说，在一个线程调用pthread_cond_wait(condvar,mutex)阻塞在condvar上，到pthread_cond_wait返回或者执行cancellation cleanup这期间（就是线程被唤醒，但是还没有从pthread_cond_wait()返回），这个mutex和condvar的绑定关系可能会被移除或者替换。尽管如此，该线程也应该重新获取这个mutex，这是pthread_cond_wait做的最后一件事情，后面代码可以看到。
5. 一个线程等待一个condvar，那么也可以说这个线程处于一个 **cancellation point**。如果该线程的**cancelability**类型被设置为PTHREAD_CANCEL_DEFERRED，那么这个线程在响应一个取消请求时，就会先重新获取mutex，再调用第一个cancellation cleanup handler。效果上就好像这个线程被唤醒，可以执行到pthread_cond_wait()返回前，然后不返回，而是开始执行线程的取消活动，这包括执行取消清理函数。
6. 一个调用pthread_cond_wait()而阻塞的线程，如果被一个取消请求唤醒，它就应该不能再消费任何条件信号。
7. pthread_cond_timedwait()和 pthread_cond_wait()基本一样。但是如果是发生超时，而不是被条件信号唤醒，那情况稍有不同。pthread_cond_timedwait会释放mutex，并重新获取mutex，而且要消费一个条件信号。
8. 如果一个正在等待一个condvar的线程收到一个信号，当 **信号处理程序返回** 时，线程应该恢复到等待condvar的状态，就好像它从没被信号中断。另外，此时pthread_cond_wait也可能返回，这也属于前面说的“伪唤醒”。

## __pthread_cond_wait_common 代码
```cpp
static __always_inline int
__pthread_cond_wait_common (pthread_cond_t *cond, pthread_mutex_t *mutex,
    const struct timespec *abstime)
{
  const int maxspin = 0;
  int err;
  int result = 0;
  LIBC_PROBE (cond_wait, 2, cond, mutex);

  uint64_t wseq = __condvar_fetch_add_wseq_acquire (cond, 2);

  unsigned int g = wseq & 1;
  uint64_t seq = wseq >> 1;

  unsigned int flags = atomic_fetch_add_relaxed (&cond->__data.__wrefs, 8);
  int private = __condvar_get_private (flags);

  err = __pthread_mutex_unlock_usercnt (mutex, 0);
  if (__glibc_unlikely (err != 0))
    {
      __condvar_cancel_waiting (cond, seq, g, private);
      __condvar_confirm_wakeup (cond, private);
      return err;
    }

  unsigned int signals = atomic_load_acquire (cond->__data.__g_signals + g);
  do
    {
      while (1)
        {
          unsigned int spin = maxspin;
          while (signals == 0 && spin > 0)
            {
              if (seq < (__condvar_load_g1_start_relaxed (cond) >> 1))
                goto done;
              signals = atomic_load_acquire (cond->__data.__g_signals + g);
              spin--;
            }
          if (signals & 1)
            goto done;
          if (signals != 0)
            break;

          atomic_fetch_add_acquire (cond->__data.__g_refs + g, 2);
          if (((atomic_load_acquire (cond->__data.__g_signals + g) & 1) != 0)
              || (seq < (__condvar_load_g1_start_relaxed (cond) >> 1)))
            {
              __condvar_dec_grefs (cond, g, private);
              goto done;
            }
          // Now block.
          struct _pthread_cleanup_buffer buffer;
          struct _condvar_cleanup_buffer cbuffer;
          cbuffer.wseq = wseq;
          cbuffer.cond = cond;
          cbuffer.mutex = mutex;
          cbuffer.private = private;
          __pthread_cleanup_push (&buffer, __condvar_cleanup_waiting, &cbuffer);
          if (abstime == NULL)
            {
              /* Block without a timeout.  */
              err = futex_wait_cancelable (
                  cond->__data.__g_signals + g, 0, private);
            }
          else
            {
              /* Block, but with a timeout.
                 Work around the fact that the kernel rejects negative timeout
                 values despite them being valid.  */
              if (__glibc_unlikely (abstime->tv_sec < 0))
                err = ETIMEDOUT;
              else if ((flags & __PTHREAD_COND_CLOCK_MONOTONIC_MASK) != 0)
                {
                  /* CLOCK_MONOTONIC is requested.  */
                  struct timespec rt;
                  if (__clock_gettime (CLOCK_MONOTONIC, &rt) != 0)
                    __libc_fatal ("clock_gettime does not support "
                                  "CLOCK_MONOTONIC");
                  /* Convert the absolute timeout value to a relative
                     timeout.  */
                  rt.tv_sec = abstime->tv_sec - rt.tv_sec;
                  rt.tv_nsec = abstime->tv_nsec - rt.tv_nsec;
                  if (rt.tv_nsec < 0)
                    {
                      rt.tv_nsec += 1000000000;
                      --rt.tv_sec;
                    }
                  /* Did we already time out?  */
                  if (__glibc_unlikely (rt.tv_sec < 0))
                    err = ETIMEDOUT;
                  else
                    err = futex_reltimed_wait_cancelable
                        (cond->__data.__g_signals + g, 0, &rt, private);
                }
              else
                {
                  /* Use CLOCK_REALTIME.  */
                  err = futex_abstimed_wait_cancelable
                      (cond->__data.__g_signals + g, 0, abstime, private);
                }
            }
          __pthread_cleanup_pop (&buffer, 0);
          if (__glibc_unlikely (err == ETIMEDOUT))
            {
              __condvar_dec_grefs (cond, g, private);

              __condvar_cancel_waiting (cond, seq, g, private);
              result = ETIMEDOUT;
              goto done;
            }
          else
            __condvar_dec_grefs (cond, g, private);
          /* Reload signals.  See above for MO.  */
          signals = atomic_load_acquire (cond->__data.__g_signals + g);
        }
    }

  while (!atomic_compare_exchange_weak_acquire (cond->__data.__g_signals + g,
                                                &signals, signals - 2));

  uint64_t g1_start = __condvar_load_g1_start_relaxed (cond);
  if (seq < (g1_start >> 1))
    {
      if (((g1_start & 1) ^ 1) == g)
        {
          unsigned int s = atomic_load_relaxed (cond->__data.__g_signals + g);
          while (__condvar_load_g1_start_relaxed (cond) == g1_start)
            {

              if (((s & 1) != 0)
                  || atomic_compare_exchange_weak_relaxed
                       (cond->__data.__g_signals + g, &s, s + 2))
                {
                  futex_wake (cond->__data.__g_signals + g, 1, private);
                  break;
                }
              /* TODO Back off.  */
            }
        }
    }
 done:
  __condvar_confirm_wakeup (cond, private);
  /* Woken up; now re-acquire the mutex.  If this doesn't fail, return RESULT,
     which is set to ETIMEDOUT if a timeout occured, or zero otherwise.  */
  err = __pthread_mutex_cond_lock (mutex);
  /* XXX Abort on errors that are disallowed by POSIX?  */
  return (err != 0) ? err : result;
}

``` 

## 实现分析
先列出源码注释中的几个重要的点。
1. pthread_cond_wait()做的事情主要是三个原子操作：（1）释放mutex并且阻塞线程；（2）unblocking；（3）重新获取mutex。该实现保证了程序中对signal和broadcast的调用，以及前面三个原子操作之间，有 **happens-before关系**。这里顺便强调一下，happens-before关系，跟时间没有任何关系。举例来说，我们程序里面有两个操作A和B，它们的关系是A happens-before B。这并不是说A一定会比B先执行，happens-before 关系不是一个时间概念上的关系，而是与内存访问有关。如果A在内存中写入了一些数据，如果B被执行的时候从那块内存中读出数据，那么B读到的就是被A更新过的数据。这与时间无关，如果B根本没有访问那块内存，那么有可能编译器或者处理器会重排序，使得B比A先执行。
2. 所有的等待线程（waiters）会从等待队列中得到一个位置。等待队列是一个64bit的序列（__wseq）。这个序列 __wseq 决定了哪些等待线程被允许消费条件信号。
3. 当一个条件信号到达时，该信号取出 __wseq 的当前值（也就是下一个等待线程将要获得的位置）。取 __wseq 的值时，指定了 relaxed-MO（relaxed memory order）。只有位置比取出的 __wseq 值还小的等待线程，才有资格消费当前信号。
4. 实现中使用了多个futex（fast user-space locking）来阻塞一系列的线程并使它们按顺序被唤醒。
5. 

























































































