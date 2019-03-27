---
layout:         page
title:         【JavaScript】闭包：实现及应用
subtitle:       
date:           2019-03-26
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 这里有必要提一笔：JavaScript权威指南（第六版）关于作用域链和闭包，有几处翻译得不恰当，实在妨碍理解。当然，这么一本大部头，翻译工作的难度可想而知。在此对翻译此书的前辈们表示敬意。

## 闭包
想要弄清楚闭包，必须先弄清楚JavaScript的变量作用域。这包括以下几个概念：词法作用域、作用域链、作用域对象。

### 词法作用域（Lexical Scoping）
JavaScript使用词法作用域：这表示JavaScript函数执行时依赖函数被定义时的变量作用域，而不是函数被调用执行时的变量作用域。比如下面的代码：
```JavaScript
var msg = "global";
var getMsg = function(){
  var msg = "scope";
  return function() {
    return msg;
  }
}();

console.log(getMsg());

// output：
scope


```
咋一看，这里很奇怪：getMsg变量引用的函数被调用时，执行return msg，而当前环境下，msg应该是"global"，结果它返回的是"scope"。原因就在于JavaScript使用的是上面说的“**词法作用域**”。因为使用了词法作用域，在JavaScript中调用函数时就不能认为仅仅是把函数体放在当前环境下来执行，函数调用还涉及变量作用域。

JavaScript通过作用域链来实现词法作用域。

### 作用域链（Scope Chain）、作用域对象（Scope Object）
如果将局部变量看作是某种由实现定义的对象的属性的话，我们就能用另一种方式来看待变量作用域：作用域链。

每段JavaScript代码（全局代码或函数体）都关联了一个**作用域链**。这个作用域链由一系列**作用域对象**组成，其中每个作用域对象包含了这段代码能访问的一个作用域（中的局部变量）。当JS要查找一个变量x时，它沿着作用域链，由近及远地在每一个作用域对象中查找其是否有一个x属性。整个作用域链中都没有找到一个包含x属性的作用域对象的话，就报错。

接下来详细说明。

#### 顶层的JS代码的作用域链
对于顶层的JS代码（即不在任何函数内的代码），其作用域链只包含一个作用域对象：全局对象。实际上，我们可以看到：在顶层代码中用var定义一个变量，这个变量就会被作为全局对象的属性。

```JavaScript
var chainX = "chainx";
let chainY = "chainy";

console.log(this);

// output

Window {postMessage: ƒ, blur: ƒ, focus: ƒ, close: ƒ, parent: Window, …}
    Iframe: ƒ (a,b,c,d,e,f,g)
    IframeBase: ƒ (a,b,c,d,e,f,g,k)
    IframeProxy: ƒ (a,b,c,d,e,f,g)
    IframeWindow: ƒ (a,b,c,d,e,f,g)
    ToolbarApi: ƒ ()
    W_jd: {}
    alert: ƒ alert()
    applicationCache: ApplicationCache {status: 0, oncached: null, onchecking: null, ondownloading: null, onerror: null, …}
    atob: ƒ atob()
    blur: ƒ ()
    btoa: ƒ btoa()
    caches: CacheStorage {}
    cancelAnimationFrame: ƒ cancelAnimationFrame()
    cancelIdleCallback: ƒ cancelIdleCallback()
    captureEvents: ƒ captureEvents()
    chainX: "chainx"
    chrome: {loadTimes: ƒ, csi: ƒ, embeddedSearch: {…}}

```
上面的代码表明，顶层代码中用var定义的变量，确实被添加到全局对象中作为属性了。但是用let定义的变量就没有。

#### 顶层函数（非嵌套函数）的作用域链
顶层函数的作用域链包含两个作用域对象：全局对象和函数局部作用域对象。这个函数局部作用域对象保存了函数自己的局部变量（包括函数实参）。

#### 嵌套函数的作用域链
嵌套函数的作用域链至少有三个对象。要弄清楚嵌套函数的作用域链，就得弄明白作用域链是怎么创建的。

**作用域链的创建：** 当**定义**一个函数时，函数就保存了一个作用域链。当该函数被调用时，它创建一个新的作用域对象并且把它添加到保存下来的那个作用域链中，这就形成了**函数调用作用域链**。

当函数被嵌套定义时，情况就更有趣了。因为每次调用外部函数时，内部嵌套的函数都会被定义一次！因为每次调用外部函数的作用域链可能是不一样的，这就导致每次定义嵌套函数时，它保存的那个作用域链也可能是不一样的。

理解作用域链的关键在于弄清两点：
1. 定义函数时，函数保存了（或者说关联了）一个作用域链。这个作用域链是定义函数的那块代码能访问的变量作用域。
2. 调用函数时，函数创建了一个新的保存了局部变量的作用域链对象，添加到上述第1点的作用域链中，形成了一个更长的函数调用作用域链。这个作用域链是函数被调用时，其函数体内可以访问的变量作用域。

接下来就可以说闭包了。

### 闭包（Closure）
词法作用域的概念上面已经说过了。为了实现词法作用域，JavaScript函数对象的内部状态不仅要包含函数的代码逻辑，还必须引用当前作用域链。函数体内的局部变量可以**保存**在作用域内（实际实现是**保存在函数作用域链中的作用域对象的属性中**），这种特性称为“闭包”。意思是好像函数把局部变量“包裹”起来了。

#### 闭包的实现
理解闭包，也就是词法作用域的规则，简单来说就是：**函数定义时的作用域（链）到函数执行时依然有效**。很明显，作用域链并不是按照栈的规则来管理的，否则，当外部函数返回时，它创建的作用域对象就该被销毁了，那嵌套函数执行时还怎么能访问到外部函数的变量作用域？

作用域链是一个链表形式的结构，并不是栈。每次调用一个函数时，都会创建一个作用域对象来绑定局部变量，并且把它添加到作用域链中。当函数返回时，这个作用域对象就被从作用域链中移除。

如果该函数没有嵌套定义函数，也就是该作用域对象没有其他的引用了，它就会被垃圾回收。

如果有嵌套的函数定义，那每个嵌套函数都会有自己的一个作用域链，并且，这个作用域链又会引用前面说的外部函数创建的那个作用域对象。

如果所有嵌套函数对象只是在外部函数内部存在的话，那这些嵌套函数也会被垃圾回收。此时，没有指向外部函数创建的作用域对象的引用，因此该对象也会一起被垃圾回收。

然而，如果外部函数把嵌套函数对象返回出去或者保存在某个外部对象的属性中，这就会有外部的引用指向嵌套函数对象。那嵌套函数对象就不会随着外部函数返回而被垃圾回收，同时，外部函数被调用时创建的作用域对象，也不会被垃圾回收，尽管创建它的函数已经返回。

这就是为什么外部函数返回了，内部的嵌套函数依然能够访问到外部函数的局部变量。

#### 闭包的应用
闭包的核心概念还是那句：**函数定义时的作用域（链）到函数执行时依然有效**。据此，我们可以使用闭包实现带缓存的函数，这种缓存技巧叫做“记忆”。

memorize函数接收一个函数参数，并返回该参数函数的带有缓存能力的版本。
```JavaScript

function memorize(func) {
    let cache = {};
    console.log("cache:");
    console.log(cache);
    return function() {
       let key = arguments.length + "," + Array.prototype.join.call(arguments,",");
       if (key in cache) {
           console.log("result is found in cache, key=" + key + ", cache=" + cache[key]);
           return cache[key];
       } else {
           return cache[key] = func.apply(this, arguments);
       }
    }
}

```
下面是使用上述memorize函数给一个求和的函数生成带缓存能力的版本的代码：
```JavaScript

var memorizeAdd = memorize(function(a, b) {
    return a + b;
});

console.log("1+2=" + memorizeAdd(1,2));
console.log("12+3=" + memorizeAdd(12,3));
console.log("1+2=" + memorizeAdd(1,2));

// output
cache:
{}
1+2=3
12+3=15
result is found in cache, key=2,1,2, cache=3
1+2=3

```
第二次使用相同参数调用memorizeAdd函数时，它从缓存中找到了结果，可以直接返回结果而不用再次计算。我们这里可以再加几行代码，会很有趣：
```JavaScript

var memorizeProduct = memorize(function(a, b) {
    return a * b;
});

console.log("1+2=" + memorizeAdd(1,2));

// output
cache:
{}
result is found in cache, key=2,1,2, cache=3
1+2=3

```
这里再次调用了memorize函数返回了一个带缓存功能的求积函数。再次调用memorize函数会再次创建一个新的作用域对象，其属性中保存了cache，并且cache此时是空对象。但是随后我们再次调用memorizeAdd(1,2)时，发现它依然在缓存中找到了结果。

原因上面已经说过：
第一次调用memorize函数返回memorizeAdd时，创建了一个作用域对象A，A的属性保存了cache。因为memorizeAdd会持有对嵌套函数对象的引用，而嵌套函数对象的作用域链又会持有作用域对象A的引用。因此作用域对象A不会被垃圾回收，尽管第二次调用memorize函数又创建了一个新的作用域对象B（它保存了另一个cache）这个作用域对象B可以被memorizeProduct访问，因为第二次调用memorize函数又**重新定义**了一个嵌套函数对象，它被memorizeProduct引用。