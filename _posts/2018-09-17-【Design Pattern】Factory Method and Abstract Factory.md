---
layout:     page
title:      【Design Pattern】Factory Method and Abstract Factory
subtitle:   Factory Method and Abstract Factory - definitions and differences.
date:       2018-09-17
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 前言

>总是在工厂方法和抽象工厂之间模糊不清，这次参考了几篇帖子，总结出两者的概念和不同。主要参考的有以下几篇：[Articles 1](https://dzone.com/articles/factory-method-vs-abstract), [Articles 2](https://stackoverflow.com/questions/5739611/differences-between-abstract-factory-pattern-and-factory-method), [Articles 3](https://www.codeproject.com/Articles/35789/Understanding-Factory-Method-and-Abstract-Factory)。

## Definitions
先看看来自《GOF》的定义。  
**Factory Method:** Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.  
**Abstract Factory:** Provide an interface for creating families of related or dependent objects without specifying their concrete classes.  
仔细分析上面的定义，能得出一个结论是，两者最大的不同之处在于，他们的关键点是 **继承(inheritance)** 还是 **组合(composition)**。后面结合代码，可以更明显地看出这一点。

## 不用任何工厂模式的代码
先看看没有用工厂模式的代码，会有什么问题。假设要解决的问题如下：我们要创建一个UI框架，由 Data, View, 以及与 View 交互的 Toolbar组成。示例代码如下：  
```cpp
class CUIFrameWork{
public:
    CUITemplate* CreateUI()
    {
        CDataComponent* pData = new CDataComponent();
        CUIComponent* pUI = new CUIComponent();
 
        CToolBarComponent* pTooBar1 = new CToolBarComponent();
        CToolBarComponent* pTooBar2 = new CToolBarComponent();
        pUI->AddToolBar(pTooBar1);
        pUI->AddToolBar(pTooBar2); 
        return new CUITemplate( pData, pUI );
    }
};
```
上面的代码有什么问题？试想一下，现在我们打算替换老的UI：替换旧有的View(CUIComponent)为一种可滚动的View(CUIComponentScrolling)。我们可能会做出如下改动：  
```cpp
class CUIFrameWork 
{
public:
    CUITemplate* CreateUI()
    {
        //与上面示例代码相同
        
    }
 
    CUITemplate* CreateScrollingUI()
    {
        CDataComponent* pData = new CDataComponent();
        CUIComponent* pUI = new CUIComponentScrolling();
    
        CToolBarComponent* pTooBar1 = new CToolBarComponent();
        CToolBarComponent* pTooBar2 = new CToolBarComponent();
        pUI->AddToolBar(pTooBar1);
        pUI->AddToolBar(pTooBar2); 
        return new CUITemplate( pData, pUI );
    }
};
```
分析一下上面的代码，至少有以下几个问题：  
1. 代码复用性太差，旧得CreateUI()中的代码没有任何复用，而新增的CreateScrollingUI()的逻辑其实与CreateUI()基本一致。
2. UI Framework 和 UI Components 具体类之间是强耦合(tightly coupled)关系：CUIFrameWork 不得不改动，以便替换一个具体的 UI Component。

## 采用工程模式的解决方案
### 采用 Factory Method
当应用工厂方法模式时，我们将创建具体组件对象的代码抽象成一个虚函数。现在，主要的框架类 CUIFrameWork，只处理创建组件的接口，实际创建组件的任务，交给 CUIFrameWork 的子类的实例。这就是上面《GOF》定义中的“Factory Method lets a class **defer** instantiation to subclasses”：真正的对象创建工作被延迟。现在，我们的代码可能是这样的：  
```cpp
class CUIFrameWork 
{
public:
    // 不再用硬编码的方式在这里创建对象，而是定义几个工厂方法

    virtual CDataComponent* MakeDataComp()
    {
        return new CDataComponent(); 
    }
 
    virtual CUIComponent* MakeUIComp()
    {
        return new CUIComponent();
    } 
    
    virtual CToolBarComponent* MakeToolBarComp( UINT nID )
    {
        return new CToolBarComponent( nID );
    } 
 
    CUITemplate* CreateUI()
    {
        CDataComponent* pData = MakeDataComp();
        CUIComponent* pUI = MakeUIComp();
        CToolBarComponent* pTooBar1 = MakeToolBarComp( ID_STANDARD );
        CToolBarComponent* pTooBar2 = MakeToolBarComp( ID_CUSTOM );
        pTooBar2->AddDropDownButton();
        pTooBar2->AddComboBox();

        pUI->AddToolBar(pTooBar1);
        pUI->AddToolBar(pTooBar2);

        return new CUITemplate( pData, pUI );
    }
};
```
现在，我们看看怎么完成我们的需求：替换一个可滚动的View。代码可能如下：  
```cpp
class CUIFrameWork_ScrollingUI : public CUIFrameWork
{
public:
    virtual CUIComponent* MakeUIComp()
    {
        return new CUIComponentScrolling();
    }
};
```
很明显，业务需求的改动引起的原有代码的改动量很小，原有代码的复用性也比较好。**另外，我们可以看到，工厂方法模式的确是使用了继承。**
#### Factory Method 的优点
1. Factory methods 消除（减小）了某些耦合。
2. 经过抽象出创建对象的接口，代码可以处理任意用户定义的具体类：只需要派生出一个框架子类，并重写相关的创建具体对象（示例中的具体的 UI Component）的接口。
3. Factory methods 提供了 **“hook”** 的机制。如果我们需要一个自定义的行为，派生一个Framework类即可。Factory methods 的 “hook”机制，在**Duilib**中也是常用的，想想看，控件是怎么绘制的，以及为什么我们可以控制如何去绘制一个控件的bkcolor, bkimage, border等等。

#### Factory Method 的缺点
1. 很明显，我们不得不派生一个类来处理变化的需求，而这个变化可能仅仅只是需要创建一个特定的具体类对象（比如我们这里的 UI Component）。也就是说，业务的变化会引起我们的系统中类的数目增加。
2. 应用了工厂方法模式以后，我们的客户代码和创建对象的“工厂”是绑定在一起的。也就是说，想通过不同的工厂来创建对象，会很困难。

### 采用 Abstract Factory
当应用 Abstract Factory 模式时，我们给系统增加的不是一个或几个method，而是一个或几个class。一个抽象工厂类包含几个工厂方法。通常，可以这么说，一个Abstract Factory是一个 Factory Method 集合，每个 Factory Method 负责创建一个具体的对象，因此，一个 Factory 对象就能创建一个对象族。每个具体的工厂类需要派生自这个抽象工厂，并实现那些工厂方法。可见，Abstract Factory 是在 Factory Method 的基础上再抽象而来。

若应用抽象工厂模式到上述代码中，我们的代码可能变成下面这样。首先看看 **抽象工厂类** 以及 **具体工厂类** 的定义：  
```cpp
//我们的 Abstract Factory 类

class CFrameWorkFactory_Abs 
{
public:
    virtual CDataComponent* MakeDataComp() = 0;
    virtual CUIComponent* MakeUIComp() = 0;
    virtual CToolBarComponent* MakeToolBarComp( UINT nID ) = 0;
};

//我们的具体的 Factory 类

class CFrameWorkFactory : public CFrameWorkFactory_Abs 
{
    virtual CDataComponent* MakeDataComp()
    {
        return new CDataComponent(); 
    }
 
    virtual CUIComponent* MakeUIComp()
    {
        return new CUIComponent();
    } 
    
    virtual CToolBarComponent* MakeToolBarComp( UINT nID )
    {
        return new CToolBarComponent( nID );
    } 
};   
```
接下来看看，工厂类怎么被应用到我们的代码中：  
```cpp
class CFrameWorkFactory_ScrollingUI : public CFrameWorkFactory
{
public:
    virtual CUIComponent* MakeUIComp()
    {
        return new CUIComponentScrolling();
    }
};

class CUIFrame 
{
public:
    CUITemplate* CreateUI(CFrameWorkFactory_Abs& objFactory)
    {
        CDataComponent* pData = objFactory.MakeDataComp();
        CUIComponent* pUI = objFactory.MakeUIComp();

        CToolBarComponent* pTooBar1 = objFactory.MakeToolBarComp( ID_STANDARD );
        CToolBarComponent* pTooBar2 = objFactory.MakeToolBarComp( ID_CUSTOM );
        pTooBar2->AddDropDownButton();
        pTooBar2->AddComboBox();

        pUI->AddToolBar(pTooBar1);
        pUI->AddToolBar(pTooBar2);

        return new CUITemplate( pData, pUI );
    }
};

```
如上，如果我们想要改变UI框架的行为，只需传入一个不同的工厂对象。

#### Abstract Factory 模式的优点
1. 抽象工厂模式隔离开了客户代码和具体类（如上面的 UI Component 类）。
2. 扩展一族具体类会很简单，只需从 Abstract Factory 派生一个具体的Factory类。客户代码改动量很少。
3. 抽象工厂模式提高了系统中那些具体类之间的一致性：每个具体的Factory类会维护好它那一族具体类之间的关系。

#### Abstract Factory 模式的缺点
1. 增加一个具体类（如上面的 UI Component 类）会比较困难：抽象工厂类需要增加接口，而系统中所有的具体工厂类全部要新增一个接口实现。


## 总结：不同
1. Abstract Factory 应用了 Factory Method 模式。从抽象层次上看，Abstract Factory 是在 Factory Method 的抽象基础上，再进行一次抽象。
2. **继承**还是**组合**，这是针对客户代码来看的。在 Factory Method 模式中，客户代码自己就充当一个工厂，它定义了工厂方法。如要使用不同的具体类，就需要
定义一个包含工厂方法的类的子类（**继承**）。而在 Abstract Factory 模式中，客户代码依赖一个工厂对象来创建具体类（**组合**），而不是自己的抽象出来的几个方法。













