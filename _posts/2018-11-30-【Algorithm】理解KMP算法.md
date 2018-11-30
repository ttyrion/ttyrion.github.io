---
layout:         page
title:         【Algorithm】理解KMP算法
subtitle:       
date:           2018-11-24
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 最近看了两篇博客，把KMP算法讲的比较清楚，但对于喜欢问“为什么”的我，其中间有一些直接给的结论我忍不住问“为什么是这样？”，当然，这个问题还得自己来回答。这篇博客就主要是记录我自己的理解以及回答自己几个“为什么”。最后会贴上上述两篇博客的链接，都是神人，能把KMP讲的比算法书还清晰。

### Naive-String-Match
朴素字符串匹配最简单最直接的匹配算法：既然要在一个字符串S中找到匹配模式串P（假设前提S.length >= P.length）的位置，那就把S中的每个可能的位置（只有前面 S.length-P.length+1 个位置才有可能是我们要的）都进行匹配测试。
```C
int naive_find_first_of(const char *str, const char *pattern) {
    const int sn = strlen(str);
    const int pn = strlen(pattern);
    int si = 0;
    
    /* Only the first (S.length-P.length+1) position maybe the solution. */
    for(; si <= sn - pn; ++si) {
        int pi = 0;
        for(; pi < pn; ++pi) {
            if(str[si+pi] != pattern[pi]) {
                break;
            }
        }

        /* match */
        if (pi == pn) {
            return si;
        }
    }

    return -1;
}
```

### Naive-String-Match 为什么会低效？
上面的Naive-String-Match算法的问题在于太“Naive”，埋头苦干（匹配），却没有利用每次匹配得到的信息。什么信息？下面就来说说。

比如下面的一个字符串匹配过程：
![naivesort1](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/algorithm/kmp/naivesort1.png) 
匹配si=6，pi=4时，发现 S[si+pi] != P[pi]，按照朴素匹配算法，则开始下一轮匹配，即si=7,pi=0,如下图：
![naivesort2](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/algorithm/kmp/naivesort2.png) 

实际上，新的S[si+pi]和P[pi]肯定不能匹配，因为在之前的匹配中，已经知道S[si+pi]=S[7]='D'，且P[pi]=P[0]='C'。**Naive-String-Match** 没有利用这些已获知的信息。

### Knuth–Morris–Pratt algorithm
应用KMP算法时，上面第一幅图中匹配失败时，会进行如下匹配：
![naivesort3](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/algorithm/kmp/naivesort3.png) 

仔细对比，在KMP算法中，并非简单地把si自增1，pi设置为0，开始新一轮匹配。KMP算法根据已获知的信息，直接开始si=8，pi=2的匹配过程。**这个过程怎么发生的？** 就是我这里要说的重点。

**注意：** 很多讲KMP算法的资料都说KMP在匹配失败时，不会改变字符串S的遍历过程，仅仅只是**移动** 模式串。这个说法是对的！我上面的匹配图中，好像si被改了，si从6变为8！实际上，对字符串S的遍历依然是不变的。因为我们访问的字符串S的位置为si+pi = 8+2 = 10。在前面匹配失败时，si+pi = 6 + 4 = 10。很明显，我的说法和他们是一样的，字符串S的索引完全没变，我们接下来还是会匹配S[10]（和P[pi]）。

那么为什么我要画这样的图，看起来好像跟其他地方讲的不一样? 其实我只是为了突出 **我们要求的解就是某个si**。我的代码也会着重于这一点，我认为算法都是一样的，代码可以有些变化。

现在匹配字符串的问题已经转化为：“在匹配失败的时候，模式串应该向右移动多少？” 这个问题其实就是想问“匹配失败的时候，应该如何改变pi的值，用新的P[pi]再去和之前的S[10]去匹配？” 这个问题用一个式子描述就是：求匹配P[pi]失败时的next(pi)，可以拿P[next(pi)]去跟S[10]匹配。

我们再次观察上面那个匹配，为什么KMP算法匹配中可以把模式串向右移动两位？因为它知道移动2位后，S[8]和P[0]匹配，而S[9]和P[1]匹配，现在要做的是匹配P[2]和S[10]，也就是说在这里，next(4)=2。算法如何知道？因为从已经匹配过的信息中，它知道S[8]=P[2],S[9]=P[3]，而模式串中P[0]=P[2],P[1]=P[3]。因此，KMP算法可以直接移动模式串（2位），使得S[8]、S[9]分别与P[0]、P[1]对齐，从S[10]和P[2]开始匹配。意思就是，当某个S[n] 和P[4] 匹配失败时候，就再把S[n]和P[next(4)],即P[2]进行匹配。从图示看来，这就相当于把模式串向右移动了4-next(4)个位置。对于这一过程，我认为**最好是理解成**：在匹配S[n]和P[pi]失败时，如何求得next(pi)，使得我们可以直接拿S[n]和P[next(pi)]来匹配。注意这里说的是“直接拿S[n]”，这表示我们没有回溯字符串S的索引，即便si因为pi的变化被间接增大了。

但是还有一种情况！

如果S[n]和P[1]匹配失败了，那么可以拿P[0]再来和S[n]进行匹配。可是如果S[n]和P[0]匹配失败，又怎么办？这里明显不能再保持n不变，找一个P[next(0)]来进行匹配。此时只能把n加1，即要匹配S[n+1]和P[0]。为了依然能用next来描述这个过程，我们定**next(0)=-1**;定next(0)=-1是有用处的：这涉及从另一个角度，即“移动模式串P”来理解KMP匹配过程：

前面说最好把KMP匹配的过程理解成：在匹配S[n]和P[pi]失败时，如何求得next(pi)，使得我们可以直接拿S[n]和P[next(pi)]来匹配。我说“最好”，有两点意思，一是为了突出next的意义，它就是为了确定下一个拿来匹配的模式串P的索引。二是这个过程当然可以有另一种理解方式。但是这个“最好的理解”在前面说的定next(0)=-1的情况下，又似乎不对：我们不能拿S[n]和一个没有意义的P[-1]来匹配。此时“另一种理解方式”就有用了：我们可以把匹配S[n]和P[next(pi)]的过程看成是，当匹配S[n]和P[pi]失败时，把模式串向右**移动** pi-next(pi)个位置。当匹配S[n]和P[0]失败时，就把模式串P向右移动0-next(0)个位置，也就是移动1一个位置，这与实际匹配过程是符合的。

上面是理解KMP算法匹配过程。那么现在问题就剩下：next(pi)是多少怎么求解？

### next数组
仔细分析上述KMP匹配的过程，可以发现在匹配P[pi]失败时的next(pi)，是模式串的一个属性，与字符串S无关！因为这个next(pi)正是模式串中的子串P[0,pi-1] （表示模式串P的0至pi-1子元素）中相同的前缀和后缀的最大字符个数，这里称之为子串的“公共前缀后缀的最大字符个数”。换种说法就是，子串P[0,pi-1] 的头N个字符和尾N个字符刚好完全一致，并且头N+1个字符和尾N+1个字符不一致，那么next(pi)就等于N。举例来说，有个子串“ABACABA”，它有个前缀A和后缀A，还有前缀ABA和后缀ABA，那么它的“公共前缀后缀的最大字符个数”就是3。

至此，对于一个模式串，我们知道有一组next值，那就把它定义成一个数组，即**next数组**。并且我们知道数组的元素个数就是模式串的字符个数。我们还知道next[0]= -1。从上面的讲解中，我们还知道，next数组只跟模式串P有关。

现在我们该考虑的问题是怎么求解一个模式串剩下的next值。既然我们已经知道next[0]= -1，那么试试**数学归纳法**：

假设next[pi] = k, next[pi+1]的值是多少呢？
因为next[pi] = k， 就表示模式子串 P[0,k-1] 和 P[pi-k,pi-1] 是一致的。那么求next[pi+1]就有两种情况：
1. 如果 P[k] = P[pi]，就知道模式子串 P[0,k] 和 P[pi-k,pi] 是一致的，这是模式子串 P[0,pi] 的“公共前缀后缀的最大字符个数”，也就是next[pi+1]。所以next[pi+1] = k+1。
2. 如果 P[k] != P[pi], next[pi+1]一定是子串 P[0,k]的“公共前缀后缀的最大字符个数”。但是我这里不打算给这么一个结论就算了，我要证明一下这个结论。假设  next[pi+1] = k2，并且假设 k2 > k，即 k2 >= k+1。那么我们知道模式子串 P[0,k2-1] 和 P[[pi+1-k2,pi] 是一致的，他们的子串 P[0,k2-2] 和 P[[pi+1-k2,pi-1] 也是一致的。因为 k2 >= k+1，所以 k2-2 >= k-1。也就是P[0,k-1]是P[0,k2-1]的子串，同样，P[pi-k,pi-1] 也是 P[[pi+1-k2,pi-1] 的子串。 这又可以反推出 next[pi] 的值是子串 P[0,k2-2] 或 P[[pi+1-k2,pi-1] 的元素个数，即  next[pi]= k2-1，因为k2-1 >= k, 而 next[pi] 一定是模式子串 P[0,pi] 的公共前缀后缀的**最大**字符个数。又因为已知了next[pi]=k，所以k2-1=k，所以k2=k+1，即next[pi+1]等于 k+1。这表示模式子串 P[0,k] 和 P[pi-k,pi] 是一致的，所以P[k]=P[pi]。但是这与我们的前提假设 P[k] != P[pi] 相悖。因此前面“假设 k2 > k”是不成立的。**反证法结束。**

 **所以结论是：如果 P[k] != P[pi], k2 一定小于或等于 k，即 next[pi+1]一定小于或等于 next[pi]。**

上面的结论可以告诉我们，如果令 next[pi+1]=k2，那么k2 <= k。这表示有模式子串 P[0,k2-1] 和 P[pi-k2+1,pi] 是一致的，可以推出 P[0,k2-2] 和 P[pi-k2+1,pi-1] 是一致的。而已知 next[pi] = k，即模式子串 P[0,k-1] 和 P[pi-k,pi-1] 是一致的。因为 k2 <= k, P[0,k2-2] 一定是 P[0,k-1] 的子串，同样， P[pi-k2+1,pi-1] 一定是 P[pi-k,pi-1] 的子串。那么可以推出 P[pi-k2+1,pi-1] =  P[pi-k2+1,pi-1] = P[k-k2+1,k-1],而前面说过 P[0,k2-2] 和 P[pi-k2+1,pi-1] 是一致的，就能推出  P[0,k2-2] = P[k-k2+1,k-1]。 而 P[0,k2-2] 或 P[k-k2+1,k-1] 的正是模式子串 P[0,k-1] 的最长的公共前缀和后缀，也就是说next[k] = k2-1，因此 k2 = next[k] + 1。

 **所以进一步得出结论： 如果 P[k] != P[pi], next[pi+1] =  next[next[pi]] + 1。这是一个递归的定义。**

至此，我们可以求出一个模式串的next数组了。





















![UML图用astah UML制作](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/Cpp11/smart_pointer.png) 


### 模板基类 _Ptr_base
_Ptr_base是std::shared_ptr 和 std::weak_ptr的公共基类，它是一个模板，其实现源码摘取部分如下：
```C

template<class _Ty>
class _Ptr_base {
public:
    typedef _Ptr_base<_Ty> _Myt;
    typedef _Ty element_type;
    void _Reset0(_Ty *_Other_ptr, _Ref_count_base *_Other_rep)
    {	// release resource and take new resource
        if (_Rep != 0)
            _Rep->_Decref();
        _Rep = _Other_rep;
        _Ptr = _Other_ptr;
    }
    //swap操作包括swap引用计数对象，以及被引用对象指针
    void _Swap(_Ptr_base& _Right) _NOEXCEPT
    {	// swap pointers
        _STD swap(_Rep, _Right._Rep);
        _STD swap(_Ptr, _Right._Ptr);
    }

    void _Decref()
    {	// decrement reference count
        if (_Rep != 0)
            _Rep->_Decref();
    }

	void _Decwref()
    {	// decrement weak reference count
        if (_Rep != 0)
            _Rep->_Decwref();
    }

	void _Reset(_Ty *_Other_ptr, _Ref_count_base *_Other_rep)
	{	// release resource and take _Other_ptr through _Other_rep
		if (_Other_rep)
			_Other_rep->_Incref();
		_Reset0(_Other_ptr, _Other_rep);
	}

    template<class _Ty2>
    void _Resetw(_Ty2 *_Other_ptr, _Ref_count_base *_Other_rep)
    {	// point to _Other_ptr through _Other_rep
        if (_Other_rep)
            _Other_rep->_Incwref();
        if (_Rep != 0)
            _Rep->_Decwref();
        _Rep = _Other_rep;
        _Ptr = const_cast<remove_cv_t<_Ty2> *>(_Other_ptr);
    }
private:
    _Ty *_Ptr;
    _Ref_count_base *_Rep;
}

```
要理解shared_ptr和weak_ptr的实现细节，主要是要弄清楚两点：
1. 它们如何对被引用的对象进行引用计数？
2. 被引用的对象如何释放？我们这里是为了探讨shared_ptr和weak_ptr在多线程环境下的安全性，因此除了前面说的两点，还会考虑：
3. 引用计数和释放是否多线程安全？

#### 1、它们如何对被引用的对象进行引用计数？
要弄清楚这个问题，先分析_Ptr_base，shared_ptr和weak_ptr派生自_Ptr_base，这两个派生类模板均只改变了部分行为。
从上面的代码可以看到，_Ptr_base包含两个数据成员：被引用的对象指针_Ptr，以及一个引用计数类的对象指针_Rep。显然，我们第一个问题，跟_Rep有关。下面列出摘取的部分_Ref_count_base代码：
```C
//include <xmemory>
#define _MT_INCR(x) \
_InterlockedIncrement(reinterpret_cast<volatile long *>(&x))
#define _MT_DECR(x) \
_InterlockedDecrement(reinterpret_cast<volatile long *>(&x))

class _Ref_count_base {
private:
    virtual void _Destroy() _NOEXCEPT = 0;
    virtual void _Delete_this() _NOEXCEPT = 0;

public:
    bool _Incref_nz()
    {	// increment use count if not zero, return true if successful
        for (; ; )
        {	// loop until state is known
#if _USE_INTERLOCKED_REFCOUNTING
            _Atomic_integral_t _Count =
                static_cast<volatile _Atomic_counter_t&>(_Uses);

            if (_Count == 0)
                return (false);

            //该操作确保weak_ptr的lock操作返回true时，被引用对象一定是未被销毁的
            if (static_cast<_Atomic_integral_t>(_InterlockedCompareExchange(
                reinterpret_cast<volatile long *>(&_Uses),
                _Count + 1, _Count)) == _Count)
                return (true);

#else /* _USE_INTERLOCKED_REFCOUNTING */
            _Atomic_integral_t _Count =
                _Load_atomic_counter(_Uses);

            if (_Count == 0)
                return (false);

            if (_Compare_increment_atomic_counter(_Uses, _Count))
                return (true);
#endif /* _USE_INTERLOCKED_REFCOUNTING */
        }
    }

    void _Incref()
    {	// increment use count
        _MT_INCR(_Uses);
    }

    void _Incwref()
    {	// increment weak reference count
        _MT_INCR(_Weaks);
    }

    //注意，_Decref会调用_Decwref
    void _Decref()
    {
        // 原子操作，确保被引用对象对象只会被释放一次
        if (_MT_DECR(_Uses) == 0)
        {	// destroy managed resource, decrement weak reference count

            // 释放被引用对象，只跟强引用计数有关
            _Destroy();

            // 当两种引用计数都为0的时候，释放引用计数对象本身
            _Decwref();
        }
    }

    void _Decwref()
    {	// decrement weak reference count
        if (_MT_DECR(_Weaks) == 0)
            _Delete_this();
    }

    long _Use_count() const _NOEXCEPT
    {	// return use count
        return (_Get_atomic_count(_Uses));
    }

    // _Expired只判断强引用计数
    bool _Expired() const _NOEXCEPT
    {	// return true if _Uses == 0
        return (_Use_count() == 0);
    }

private:
    _Atomic_counter_t _Uses;  //强引用计数
    _Atomic_counter_t _Weaks; //弱引用计数

protected:
    _Ref_count_base()
    {	// construct
        _Init_atomic_counter(_Uses, 1);
        _Init_atomic_counter(_Weaks, 1);
    }
}
```
_Ref_count_base有两个成员_Uses和_Weaks，分别表示**强引用计数**（shared_ptr的引用计数类型），以及**弱引用计数**（weak_ptr的引用计数类型）。_Ref_count_base分别提供了操作（增加或减少）这两个引用计数的接口_Incref/_Decref 和 _Incwref/_Decwref。

为了说明方便，先贴出摘取的shared_ptr 和 weak_ptr 源码:
```C
// std::shared_ptr
template<class _Ty> 
class shared_ptr : public _Ptr_base<_Ty> {
public:
    typedef shared_ptr<_Ty> _Myt;
    typedef _Ptr_base<_Ty> _Mybase;

    // 构造函数，不是原子操作
    template<class _Ux> explicit shared_ptr(_Ux *_Px)
    {	// construct shared_ptr object that owns _Px
        _Resetp(_Px);
    }

    template<class _Ty2>
    explicit shared_ptr(const weak_ptr<_Ty2>& _Other,
        bool _Throw = true)
    {	// construct shared_ptr object that owns resource *_Other
        this->_Reset(_Other, _Throw);
    }

    ~shared_ptr() _NOEXCEPT
    {	// release resource
        this->_Decref();
    }

    void reset() _NOEXCEPT
    {	// release resource and convert to empty shared_ptr object
        shared_ptr().swap(*this);
    }

    template<class _Ux> void reset(_Ux *_Px)
    {	// release, take ownership of _Px
        shared_ptr(_Px).swap(*this);
    }

    void swap(_Myt& _Other) _NOEXCEPT
    {	// swap pointers
        this->_Swap(_Other);
    }

    // 注意内部调勇的是move，因此参数_Right会变成无效对象
    _Myt& operator=(_Myt&& _Right) _NOEXCEPT
    {	// take resource from _Right
        // 调勇了swap，显然也不是原子操作
        shared_ptr(_STD move(_Right)).swap(*this);
        return (*this);
    }

    _Myt& operator=(const _Myt& _Right) _NOEXCEPT
    {	// assign shared ownership of resource owned by _Right
        shared_ptr(_Right).swap(*this);
        return (*this);
    }

private:
    template<class _Ux> void _Resetp(_Ux *_Px) {	// release, take ownership of _Px
        _TRY_BEGIN	// allocate control block and reset
            _Resetp0(_Px, new _Ref_count<_Ux>(_Px));
        _CATCH_ALL	// allocation failed, delete resource
            delete _Px;
        _RERAISE;
        _CATCH_END
    }

    template<class _Ux>
    void _Resetp0(_Ux *_Px, _Ref_count_base *_Rx)
    {	// release resource and take ownership of _Px
        this->_Reset0(_Px, _Rx);
        _Enable_shared(_Px, _Rx);
    }
};
```



下面就可以从几种场景中来看引用计数变化的情况，以shared_ptr为例：
1. 构造智能指针对象。其构造函数shared_ptr<_Ty>::shared_ptr(_Ux *_Px) 会调用shared_ptr<_Ty>::_Resetp(_Ux *_Px)，这里会new一个 _Ref_count<_Ty>对象，而其基类_Ref_count_base<_Ty>的构造函数中，会把强引用计数_Uses，以及弱引用计数_Weaks的值都设置为1。其他构造函数也类似。
2. 智能指针对象赋值。赋值运算时，会执行shared_ptr(_Right).swap(*this);这里复制构造了一个临时的shared_ptr<_Ty>对象，再执行了swap操作。第一步复制构造时，会调用_Ptr_base<_Ty>_Reset(_Ty *_Other_ptr, _Ref_count_base *_Other_rep);这里会增加赋值运算过程的源对象的引用计数，接着调用_Reset0，这里减少目标智能指针对象的引用计数。
3. 智能指针对象释放。shared_ptr<_Ty>对象释放时，执行this->_Decref(); 前面已经说过，这会减少_Ref_count_base引用计数对象_Rep中的强引用计数。

weak_ptr的这个过程，跟shared_ptr基本一样，不再赘述。这里仅贴出摘取的weak_ptr代码：
```C
// std::weak_ptr
template<class _Ty>
class weak_ptr
    : public _Ptr_base<_Ty> {
public:
    typedef weak_ptr<_Ty> _Myt;
    typedef _Ptr_base<_Ty> _Mybase;

    weak_ptr(const weak_ptr& _Other) _NOEXCEPT
    {	// construct weak_ptr object for resource pointed to by _Other
        this->_Resetw(_Other);
    }
    weak_ptr(_Myt&& _Other) _NOEXCEPT
        : _Mybase(_STD move(_Other))
    {	// move construct from _Other
    }

    ~weak_ptr() _NOEXCEPT
    {	// release resource
        this->_Decwref();
    }

    weak_ptr& operator=(const weak_ptr& _Right) _NOEXCEPT
    {	// assign from _Right
        this->_Resetw(_Right);
        return (*this);
    }

    shared_ptr<_Ty> lock() const _NOEXCEPT
    {	// convert to shared_ptr
        return (shared_ptr<_Ty>(*this, false));
    }
};

```

前面分析了对被引用对象进行计数的情况，接下来分析第二个问题：

#### 2、被引用的对象如何释放？
对于这个问题，只需看_Ref_count_base::_Decref() 以及 _Ref_count_base::_Decwref()，这里再次贴出这两个函数代码：
```C
    
	//注意，_Decref会调用_Decwref
    void _Decref()
    {
        // 原子操作，确保被引用对象对象只会被释放一次
        if (_MT_DECR(_Uses) == 0)
        {	// destroy managed resource, decrement weak reference count

            // 释放被引用对象，只跟强引用计数有关
            _Destroy();

            // 当两种引用计数都为0的时候，释放引用计数对象本身
            _Decwref();
        }
    }

    void _Decwref()
    {	// decrement weak reference count
        if (_MT_DECR(_Weaks) == 0)
            _Delete_this();
    }

```
这两个函数的意义很明显：当强引用计数变为0的时候，调用_Destroy();当弱引用计数变为0的时候，调用_Delete_this()。从上面代码可以看到，这两个函数都是纯虚函数。其实现在子类中：
```C
template<class _Ty>	class _Ref_count : public _Ref_count_base {
public:
    _Ref_count(_Ty *_Px)
        : _Ref_count_base(), _Ptr(_Px)
    {	// construct
    }
private:
    virtual void _Destroy() _NOEXCEPT
    {	// destroy managed resource
        delete _Ptr;
    }

    virtual void _Delete_this() _NOEXCEPT
    {	// destroy self
        delete this;
    }

    _Ty * _Ptr;
};

```
上面的代码可以看出，_Destroy()正是销毁被引用对象_Ptr，而_Delete_this()是销毁引用计数对象本身。

这里值得指出一下几点：
1. 正在持有被引用对象（管理被引用对象）的是引用计数对象，而不是智能指针对象。
2. 被引用对象_Ty的释放，只跟强引用计数有关。弱引用计数只影响引用计数对象本身的释放。只有当两个引用计数都为0时，才释放引用计数对象本身，这就是为什么_Decref()判断强引用计数为0，调用 _Destroy()之后还要调用 _Decwref()，就是为了释放引用计数对象本身，因为该对象构造时，弱引用计数被初始化为1。

#### 3、引用计数和释放是否多线程安全？
上面贴出的代码，其实已经可以看出以下两点：
1. shared_ptr和weak_ptr的reset、swap、constructor等等操作，都不是原子的，因此在多线程环境下，必须由我们自己确保对他们的相关访问是同步的。
2. 被引用对象的释放，只会发生一次。这是因为_Ref_count_base中的引用计数的操作是原子的。但是，这不代表我们调用shared_ptr<_Ty>::get()返回的一定是有效的对象，不能混淆。这里的原子操作仅仅只是确保被引用对象_Ty不会被多次释放。

**总结一句话就是： shared_ptr和weak_ptr都不是多线程安全的，我们需要自己同步。**

### std::weak_ptr<_Ty>::lock()
```C
	// weak_ptr
    shared_ptr<_Ty> lock() const _NOEXCEPT
    {	
		// convert to shared_ptr
        return (shared_ptr<_Ty>(*this, false));
    }
};

```
如上面代码所示，weak_ptr<_Ty>的lock()调用了shared_ptr的以weak_ptr为第一个参数的构造函数，创建了一个shared_ptr<_Ty>对象。这个构造函数内部又调用了_Ptr_base<_Ty>::_Reset(_Ty *_Other_ptr, _Ref_count_base *_Other_rep, bool _Throw):
```C

// _Ptr_base
void _Reset(_Ty *_Other_ptr, _Ref_count_base *_Other_rep, bool _Throw)
{	
	// take _Other_ptr through _Other_rep from weak_ptr if not expired
	// otherwise, leave in default state if !_Throw,
	// otherwise throw exception
	if (_Other_rep && _Other_rep->_Incref_nz())
		_Reset0(_Other_ptr, _Other_rep);
	else if (_Throw)
		_THROW_NCEE(bad_weak_ptr, _EMPTY_ARGUMENT);
}

```
从上面的_Reset可以看出，这里可能发生两种情况：
1. _Other_rep，即原始的weak_ptr<_Ty>对象的引用计数对象，其引用计数不为0时，增加其引用计数，并调用_Reset0，之后lock()创建的shared_ptr<_Ty>对象会持有原始weak_ptr<_Ty>对象引用的那个_Ty对象。
2. 原始的weak_ptr<_Ty>对象的引用计数对象的引用计数为0时，_Reset什么也不做。也就是说，lock()创建的shared_ptr<_Ty>对象是一个空对象，不会持有任何被引用的_Ty对象。

所以，只需要判断lock()返回的shared_ptr<_Ty>对象是否为空对象，如非空，则它一定是引用了一个有效的_Ty对象。如：
```C

// std::weak_ptr<StoryBoard> anim_observer_;
if (!anim_observer_.expired()) {
	auto shared = anim_observer_.lock();
	if (shared.get()) {
		shared->OnAnimationStage(FADE_HIDE_ANIM, stage);
	}
}

```
上面的代码，在多线程环境中也不会有问题。内部嵌套的if中的shared，一定是引用的有效的StoryBoard对象。


### 总结
**shared_ptr和weak_ptr在不是多线程安全的，如果有多个线程读写它们我们需要同步。这里，把通过shared_ptr和weak_ptr调用被引用对象的方法的操作称为读，调用shared_ptr和weak_ptr的reset，operator= 等操作称为写。**