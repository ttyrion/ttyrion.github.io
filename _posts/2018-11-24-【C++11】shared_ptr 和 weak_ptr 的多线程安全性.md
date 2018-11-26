---
layout:         page
title:         【C++11】shared_ptr 和 weak_ptr 的多线程安全性
subtitle:       从源码看智能指针的多线程安全性
date:           2018-11-24
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 本篇主要讲述std::shared_ptr 和 std::weak_ptr在多线程中的安全性问题。

### std::shared_ptr、std::weak_ptr 的类层次结构

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