---
layout:         page
title:         【JavaScript】原型对象、对象继承以及属性查找
subtitle:       
date:           2019-03-23
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

##### 原型对象
每个JavaScript对象（除了null和Object.prototype之外）都和另一个“**对象**”关联在一起，这些JavaScript对象继承了此“另一个”对象的属性。这个“另一个对象”就称为原型对象（prototype）。

通过关键字new以及构造函数创建的对象的原型对象就是其构造函数的prototype属性值。比如说，new Object() 返回的对象，继承了Object.prototype对象的属性（包括toString等等）。

一个对象obj的原型对象，以及此原型对象的原型对象......等等，形成了所谓的“**原型链**”。在ES5中，可以通过 **Object.getPrototypeOf(obj)** 获取对象obj的原型对象。

##### 属性的查找
从上面可得知：对象obj的属性x（假设此属性存在），可能是对象obj自定义的属性（**自有属性**），也可能是继承自其原型对象，也有可能继承自其原型对象的原型对象......

JavaScript解释器访问obj的属性x时，先在对象obj的自有属性中查找是否存在x属性。若不存在，则沿着原型链向上查找每一个原型对象是否有x属性，直到找到x。若最终依然没找到属性x，那么x属性就是**undefined**。

请注意：上面这一说法，也就告诉了我们，JavaScript对象继承属性时，并没有创建属性的副本。下面的代码可以证明这一点：
```JavaScript
var base = {
    x: "base"
};

var derived = Object.create(base);
console.log(derived.x)

delete base.x;
console.log(base.x)
console.log(derived.x)

// output：
base
undefined
undefined
```
上面的代码说明了：删除了原型对象的x属性之后，我们定义的对象derived也没有了x属性。JavaScript解释器在对象derived的自有属性以及原型链中的所有原型对象中都找不到属性x，因此x就是undefined。

这告诉我们：如果修改了原型对象的属性（包括修改其属性值、删除该属性等等），可能会导致所有继承了该原型对象的对象都受到影响！看了下面的内容，就知道为什么这里说“**可能**”。

##### 属性的覆盖
对于上面的情况，考虑：如果原始对象（即derived引用的对象）和原型对象共有一个属性，岂不是很混乱？修改原始对象的属性会影响原型对象的属性，反之亦然......

这种情况当然是不可能的！因为在修改**原始对象**的x属性时，原始对象derived就会有x属性的一个“**副本**”。这叫做“**属性覆盖**”：原始对象的属性覆盖了原型对象的属性。下面的代码可以证明这一点：
```JavaScript
var base = {
    x: "base"
};

var derived = Object.create(base);
console.log(derived.x);

derived.x = "derived";
delete base.x;
console.log(base.x);
console.log(derived.x);

// output：
base
undefined
derived
```
上面的代码说明了：当修改了derived的x属性值之后，derived就有了一个x属性的副本。此后derived和base引用的对象的x属性就不会互相影响。所以即便是删除了base的x属性，derived的x属性依然可以正常访问。




