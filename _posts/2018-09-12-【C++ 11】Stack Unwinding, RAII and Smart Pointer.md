---
layout:     page
title:      【C++ 11】Stack Unwinding, RAII and Smart Pointer
subtitle:   RAII, std::auto_ptr, std::shared_ptr, std::unique_ptr, std::weak_ptr
date:       2018-09-12
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 前言

>资源管理（主要是资源释放）绝对是 C++ 开发中很重要的问题之一，弄不好就会造成内存泄漏。由此，C++也引入智能指针来解决问题。

## 原始指针的问题
原始指针的问题主要在释放，代码中可能有各种原因导致在堆中分配的内存没有被释放，最简单的比如说：  
1. 在一大块代码之后忘了 delete，不过如果我们习惯在敲完new之后紧接着敲delete，再在delete前插入其它代码的话，“忘记delete”就是一个不成问题的问题。
2. 代码执行起来，可能由于某种运行时原因，不会走到我们敲的delete。下面会举例子。  

测试代码：  
```cpp
class Test {
public:
    Test(int m) {
        m_ = m;
    }
    ~Test() {
        std::cout << "Test desturcted." << std::endl;
    }

    int m_ = 0;
};

void TestFun(int m) {
    throw std::exception("test exception.");
}

//有一段测试的代码如下

try {
    Test* pt = new Test(1);
    TestFun(pt->m_);
    delete pt;
}
catch (std::exception& e)
{
    std::cout << e.what() << std::endl;
}
```
上面的那段测试代码中，delete pt就不会被执行。这只是一个简单的例子，实际上，如果有很多地方会通过指针 pt 访问对象（这里是一个Test对象），那么这个释放就更容易出问题：
比如某个地方的代码因判断到一个意外情况而使业务逻辑并没有正常走到我们 delete pt 的地方，这就会导致内存泄漏。

## 栈回溯（stack unwinding）
代码中很多地方都在调用函数或创建局部对象（即在栈上创建对象，而不是在堆上），这种情况称为 **"stacked up"** 。  
当退出一个代码块（一个作用域）时，不论是由于 return，还是到达作用域结尾（reaching the end of the scope）, 还是有抛出了异常，作用域内的一切都会被销毁（即所有对象的析构函数会被调用）。
这个调用析构函数销毁局部对象的过程就称为 **"stack unwinding"** 。同时，因为存在 **"stack unwinding"** ，这就要求我们的类的析构函数里面不要抛出异常。原因之一是C++要求在已经被抛出的
异常被处理完之前，不能再抛出新的异常。原因之二，是在栈回溯过程中，析构函数抛出异常，会导致一种未定义的行为。也就是说，不论栈回溯是因为异常引起，还是因为离开一个作用域，我们都不能
在析构函数中抛出异常。如要一个函数不抛出异常，可以在函数后加 **noexcept** 声明，此时编译器会对函数里面的 throw 提示 **waring**。  
**"stack unwinding"** 还引出了一个C++技术，称为 **RAII**（Resource Acquisition Is Initialization），来帮助开发者管理内存资源，数据库连接，或者打开的文件描述符等等。

## RAII (Resource Acquisition Is Initialization)
RAII，即资源获取即初始化。这是 Bjarne Stroustrup 提出的一个设计理念。其核心是把资源的创建（获取）和释放与对象的生命周期绑定在一起。在RAII的指导下，C++把“资源管理”这一底层问题提升到了
“对象生命周期管理”这一更高的层次上。不过我个人认为，把资源的管理和对象的生命周期绑定在一起，也并不是说一定要在创建对象（constructor）中创建资源，在销毁对象（destructor）中释放资源。
还是应该考虑实际情况，在需要资源的时候申请资源，在对象销毁时根据实际情况判断是否有资源需要释放。  

## 智能指针 (smart pointer)
智能指针就是基于RAII的设计思想，设计出的管理动态分配资源的类。它跟RAII的思想可以说是完全吻合：构造智能指针对象的时候占有对象内存资源，智能指针对象退出作用域的时候释放对应的对象内存资源。  

### std::auto_ptr 以及为什么它没什么用
std::auto_ptr 是C++ 98 引入的。在上面的例子中，如果把测试的那段代码改为如下：  
```cpp
try {
    std::auto_ptr<Test> pt(new Test(1));
    TestFun(pt->m_);
}
catch (std::exception& e)
{
    std::cout << e.what() << std::endl;
}
```
那么在堆上分配的Test对象在栈回溯过程中就会被释放，不会有上面的问题。std::auto_ptr的特性是控制权 **转移** （transfer its ownership）。看如下测试代码:
```cpp
void TestFun(std::auto_ptr<Test> arg)  {
    std::cout << "TestFun Test::m_ = " << arg->m_ << std::endl;
}

//有一段如下的测试代码

std::auto_ptr<Test> pt(new Test(1));
TestFun(pt);
int result = pt->m_;
```
上面的代码会怎么样？crash！调用TestFun函数的时候，pt对象把堆上的Test对象的控制器转移给了TestFun函数的形参，之后，pt这个对象本身就变成了一个**“空对象”**。
因此再通过pt访问Test对象的m_时，就会崩溃。

正是因为控制权**转移**的特性，使得 std::auto_ptr 的copy语义并不是通常的复制或赋值，因此，std::auto_ptr 也不能被用于C++标准库容器（vector, list, map等等）中。
**因此我个人认为 std::auto_ptr 有自己的问题，简直是不堪重用。** C++ 11 也因此新增了三个智能指针。

### std::shared_ptr
上面曾说道，std::auto_ptr 的特性是 "transfer ownership", 相应的，std::shared_ptr 的特性是 "share ownership"。std::shared_ptr的作用很简单：多个std::shared_ptr智能指针
可以引用同一个对象，当最后一个std::shared_ptr智能指针对象离开其作用域时，它会释放其所引用的对象。上面发生crash的代码，把 std::auto_ptr 替换成 std::shared_ptr 就不会有问题。

std::shared_ptr 的引用计数，称之为 **强引用计数（strong reference）**，相对于 std::weak_ptr 的 **弱引用计数（weak reference）**。Visual Studio 调试时也会提示
std::shared_ptr 的引用计数为强引用，比如 "pt2 = shared_ptr {m_=1 } [2 strong refs] [default]"。也是因为 std::shared_ptr 的引用计数，不能用同一个

#### 资源释放 by std::shared_ptr
std::shared_ptr 默认会调用 delete 来释放其所引用的对象，但是我们也可以指定一个销毁操作。比如：
```cpp
    std::shared_ptr<Test> pt(new Test[3],
        [](Test* arg) {
            delete[] arg;
        }
    );
```
上面的代码中，shared_ptr引用的是Test对象的数组，如果仍用默认的delete去释放资源就会出问题，这个时候，可以指定一个合适的释放操作。

### std::weak_ptr
上面提到auto_ptr和shared_ptr的时候，都提到了"ownership"。不论是auto_ptr还是shared_ptr，它们都有被引用的对象的控制权（转移的或者共享的控制权），
因而auto_ptr和shared_ptr都有operator->，可以直接操作对象。

std::weak_ptr就不同了。它并没有"ownership"，只是有一个前面已提到过的弱引用计数(weak reference)。因此，weak_ptr也并没有提供 ->, * 等操作符来访问被引用的对象。
那么 std::weak_ptr 到底有什么用呢？
```cpp
void TestFun(std::weak_ptr<Test> arg)  {
    if (!arg.expired()){
        std::cout << "TestFun Test::m_ = " << arg.lock()->m_ << std::endl;
    }    
}

//一段测试代码

std::shared_ptr<Test> pt(new Test(3));
std::weak_ptr<Test> wpt(pt);
pt.reset();
TestFun(pt);

```
如上面的TestFun函数，参数arg只会增加对象的弱引用计数（weak ref count）,而不会增加对象的强引用计数（strong ref count）。也就是说，TestFun函数体只是在可能的条件下，
需要访问被引用的对象，但是却不希望影响对象的生命周期。在这种情况下，std::weak_ptr 就能排上用场。
下面分析一下上面测试代码中对象的引用计数变化情况。
```cpp
1 std::shared_ptr<Test> pt(new Test(3));
  执行完上面第一行代码后，Test对象的引用计数情况：1 strong ref count and 0 weak ref count
2 std::weak_ptr<Test> wpt(pt);
  到这里，1 strong ref count and 1 weak ref count
3 pt.reset();
  到这里，0 strong ref count and 1 weak ref count
4 TestFun(pt);
```
上面的代码执行完第3行后，Test对象就被释放了。在TestFun函数内部，判断weak_ptr引用的对象已经无效，因而不会访问对象的 m_ 成员。

### std::unique_ptr
unique_ptr 可以说跟auto_ptr很像，它也没有一般的 copy 语义，而是一种 **"move"** 语义。简单来说，就是 unique_ptr 保证了在任何时候，
都只有一个 unique_ptr 引用着对应的资源。当 unique_ptr 对象离开作用域的时候，就会释放它引用的资源。

unique_ptr 和 auto_ptr 难道是一样的？答案是，它们稍有区别。unique_ptr可以支持管理一个对象的数组，这是 auto_ptr 不支持的。
```cpp
std::unique_ptr<Test[]> upt(new Test[3]);
```
像上面那样，unique_ptr可以管理一个数组，但是不需要跟shared_ptr一样，由开发者提供一个合适的释放操作。
**注意，对于std::unique_ptr，要明确的一点是：它不提供copy语义，因而跟auto_ptr一样，不能被用在标准库容器中。**
