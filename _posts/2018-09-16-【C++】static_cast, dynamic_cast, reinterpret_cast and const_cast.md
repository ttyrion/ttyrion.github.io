---
layout:     page
title:      【C++】static_cast, dynamic_cast, reinterpret_cast and const_cast
subtitle:   几个cast的作用和区别
date:       2018-09-16
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## static_cast
static_cast 可以在一下几种情况下进行转换：  
1. 可以进行隐式类型转换的情况。如非const对象转换为const对象（反过来不行），int转换为double。
2. **void\*** 和 **T\*** 之间相互转换。
3. 父类对象的指针和子类对象的指针之间相互转换。即，static_cast可以支持 **upcast** 和 **downcast** 。子类对象转换为父类对象这一情况也属于隐式类型转换（Derived对象 **is-a** Base对象），但是反过来的情况则不属于隐式转换。  

```cpp
class A {
public:
    int x = 0x41;
    int y = 0x42;
};

class D : public A {
public:
    int z = 0x43;
};

class C {};

int main()
{
    A a;
    D d;
    A* pa1 = &d;                   //OK, implicit conversion
    
    A* pa2 = static_cast<A*>(&d);  //OK, explicit conversion, upcast
    
    //D* pd1 = &a;                 //Error! 不能隐式转换
    
    D* pd2 = static_cast<D*>(&a);  //Ok, downcast, pd2->x:65, pd2->y:66, pd2->z:-858993460 (没经过初始化的随机值)
    
    C* pc1 = (C*)(&a);             //OK, Force conversion
    
    //C* pc = static_cast<C*>(&a); //Error! static_cast不能在 unrelated 类之间进行转换

    
    return 0;
}

```
上面的注释说明了各种情况的转换。但是实际写代码时需要注意：  
1. static_cast不仅支持upcast，还支持downcast，但是没有运行时类型检查。**dynamic_cast提供安全的downcast。**
2. static_cast不仅支持把T*指针转换为void*，反过来也可以把void*转换为T*，同样也没有运行时类型检查。

## reinterpret_cast
C++的 reinterpret_cast 是低层次的转换（在二进制位层面上），并且依赖于具体实现，因此是不可移植的。  
reinterpret_cast跟C的强制类型转换(T*)(expression)很像。看这个名字中的**interpret**，什么意思？“解释”。也就是说，这个转换就是把当前某个对象S当做另一个对象D来解释，根本不管 S 和 D 是什么关系。
那么可以看出reinterpret_cast有两个特点：
1. 对参与转换的两个对象类型没有要求。注意是对类型没有要求，reinterpret_cast 也不能去掉被转换对象的const属性。
2. 因为第1点，reinterpret_cast很危险。除非很明确自己的目的，否则不应该用它。 
 
```cpp
class A {
public:
    int x = 0x41;
    int y = 0x42;
};

int main()
{
    int i = 10;
    A* pa = reinterpret_cast<A*>(&i);    //OK, pa->x : 10, pa->y : -858993460(显然，这是一个随机的，没被初始化的值)
    
    std::string* ps1 = reinterpret_cast<std::string*>(&a);  //OK
    
    std::string* ps2 = (std::string*)(&a);                  //OK
    
    std::string str = *ps;                                  //Crash
    
    
    return 0;
}

```
正如上面的示例代码，reinterpret_cast可以把A类型的对象a，转换为一个完全不相关的字符串string类型对象。这个转换不会有问题，因为reinterpret_cast不会做什么检查，也不会对被转换对象做任何改动。可以认为它只是一个说明，说明我们要把那个对象（那块内存）当做新类型的对象来处理。然而，正因为reinterpret_cast不做任何改动，仅仅只是一个“重新解释那块内存”的说明，随后对新对象的访问就会造成崩溃：因为那个对象根本不是std::string。


## dynamic_cast
dynamic_cast有运行时类型检查，用于提供安全的downcast转换：判断一个对象是否是继承层次中的一个特定类型对象。或者简单来说就是，判断一个基类对象是否是一个特定子类的对象。当然，dynamic_cast也支持upcast。**需要注意，dynamic_cast也是唯一一个有较显著的性能损失的转换行为。**
另外，**dynamic_cast并不支持所有的downcast转换，只有当这个继承层次的类支持多态时才可以。** 
1. 若dynamic_cast用于在指针之间进行转换，失败时dynamic_cast返回空指针(nullptr)。
2. 若dynamic_cast用于在引用之间进行转换，失败时会抛出bad_cast异常。

```cpp
class A {
public:
    int x = 0x41;
    int y = 0x42;
};

class D : public A {
public:
    int z = 0x43;

    virtual void cast() {}
};

class C : public D {
public:
    void cast() override {}

    int c = 0;
};

int main()
{
    A a;
    D d;
    A* pa = dynamic_cast<A*>(&d);
    //D* pd = dynamic_cast<D*>(&a);  //Error, 不是多态类型
    
    C* pc = dynamic_cast<C*>(&d);    //OK, 多态类型
    

    return 0;
}
```

## const_cast
const_cast 的使用场景比较简单，通常用于去掉对象的const属性。

## 总结
1. **reinterpret_cast** 可以在两个毫无关系的类型的对象之间进行转换。因为它是在bit层面上的转换，简单理解就是，它给我们一种方式去随意解释一块内存所属对象的类型，而不管该对象真正的类型。当然，它不能去掉对象的const属性。
2. **static_cast** 没有运行时类型检查。
3. **dynamic_cast** 有较大的性能损耗。

