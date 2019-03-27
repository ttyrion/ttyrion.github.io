---
layout:         page
title:         【JavaScript】数组的map、reduce方法分析及实现
subtitle:       
date:           2019-03-24
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

## map方法 (ES5)
如其名字所示，map方法返回一个“映射”数组，原始数组的每一个元素映射到返回的数组的对应元素。映射关系由我们传给map方法的参数函数（这里称为**映射函数**）确定。与forEach的参数一样，映射函数有三个参数：数组元素、元素索引、原始数组本身。映射函数也可以根据需求定义不超过3个形参，多余的形参会被传undefined实参。
```JavaScript

var srcArray = [1,,3,4,5];

var mapArray = srcArray.map((item,index,arr) => {
  console.log(item+ " " + index + "; " + arr);
});

console.log(mapArray);

// output
1 0; 1,,3,4,5
3 2; 1,,3,4,5
4 3; 1,,3,4,5
5 4; 1,,3,4,5
(5) [undefined, empty, undefined, undefined, undefined]

```
上面的输出可以看出四点：
1. map方法调用映射函数时也会跳过空缺元素。
2. 空缺元素和undefined是不一样的。
3. map返回的数组的元素是映射函数的返回值，因为这里的映射函数没有返回值，所以map返回的数组的元素都是undefined。
4. 空缺元素依然被保留到输出的映射数组中。

###### map方法实现
```JavaScript

if (!Array.prototype.map2) {
  Array.prototype.map2 = function (func) {
    result = [];
    const len = this.length;
    for (let i = 0; i < len; ++i) {
      if (i in arr) {
        result[i] = func.call(null, this[i], i, this);
      }
    }

    return result;
  };
}

```
map2方法也能完成类似ES5的map方法的功能。示例代码如下：
```JavaScript

var srcArray = [2,,3,4,5];
var mapArray = srcArray.map2((item,index,arr) => {
  console.log(item + " " + index + ";    " + arr);
  return item * item;
});

console.log(mapArray);

// output
2 0;    2,,3,4,5
3 2;    2,,3,4,5
4 3;    2,,3,4,5
5 4;    2,,3,4,5
(5) [4, empty, 9, 16, 25]

```

## reduce方法 (ES5)
reduce比map稍微复杂一点，reduce是“化简”方法，它把数组化简为一个值。reduce有两个参数，第一个参数是化简函数，第二个参数是初始值。要搞清楚reduce的作用，要分两部分：化简函数、reduce怎么调用化简函数。

**化简函数**：
```JavaScript

function(prev,item,index,arr) {
  ...
}

```
化简函数有四个参数：第一个参数表示前一次化简的结果，后三个参数表示数组元素、元素索引、原始数组本身。原始数组的每一个元素都会参与化简运算（**除了空缺元素**），前一次的化简结果会被reduce用来调用下一次化简函数。因此最终reduce的返回值，是一个由所有元素参与运算得出的结果。细心的朋友这里就会想到一个问题：reduce第一次调用化简函数的时候，怎么传“前一次化简的结果”给化简函数呢？这里就用到reduce第二个参数了，第二个参数就表示化简结果的初始值。如果没传第二个参数，reduce就用数组的第一个非空缺元素作为第一次调用化简函数时的第一个实参，第二个非空缺元素作为第一次调用化简函数时的第二个实参，第二个非空缺元素的索引作为第一次调用化简函数时的第三个实参。

**reduce如何调用化简函数**：经过上面对化简函数的说明，reduce调用化简函数的情形也很清晰了。
1. 如果调用数组reduce方法时传了初始值，那reduce就把初始值作为第一个实参，依次把每个数组元素作为后续实参，来调用化简函数。并且，前一次的化简结果，又作为后一次化简的第一个实参。
2. 如果调用reduce方法时没传初始值，reduce第一次调用化简函数时的情况如上面介绍化简函数中的说明。第二次以及以后的调用，就如上面第1点所描述的一样。

先拿一个没有实际意义的例子来说明reduce调用化简函数的情况：
```JavaScript

var srcArray = [1,,3,4,5];

var result = srcArray.reduce((prev,item,index,arr) => {
  console.log(prev + " " + item+ " " + index + ";       " + arr);
  return "reduce";
});

console.log(result);

// output
1 3 2;       1,,3,4,5
reduce 4 3;       1,,3,4,5
reduce 5 4;       1,,3,4,5
reduce

```
上面的例子有几点说明：
1. 调用reduce时没传初始值，所以reduce把数组第一个元素作为初始化简结果，把第三个元素（**跳过空缺元素**）作为第二个实参调用化简函数。
2. 前一次的化简结果被作为后续调用化简函数的第一个实参，因为这个化简函数每次都返回"reduce"，所以可以看到后面几次调用时前一次化简结果实参prev的值都是"reduce"。
3. 最终的化简结果是"reduce",因为最后一次调用化简函数的返回值，还是"reduce"。

这个例子很好地说明了reduce化简的过程。但是没有实际意义。下面这个计算数组元素之和的例子可以看出reduce的简洁！
```JavaScript

var srcArray = [2,,3,4,5];
var sum = srcArray.reduce((prev,item) => {
  return (prev + item);
});
console.log("sum: " + result);

// output
sum: 13

```

再来一个求数组元素的平方和的例子：
```JavaScript

var srcArray = [2,,3,4,5];
var sumOfSquare = srcArray.reduce((prev,item) => {
  return (prev + item * item);
});
console.log("sumOfSquare: " + sumOfSquare);

// output
sumOfSquare: 52

```
最后输出的结果不对！原因在哪里？就在第一次调用时。按照前面的说明，第一次调用时初始化简结果prev参数是数组第一个元素，也就是2。可见我们求数组元素平方和时，第一个元素并没有算平方，而是直接计算和了。此时就需要给reduce传一个初始值：
```JavaScript

var srcArray = [2,,3,4,5];
var sumOfSquare = srcArray.reduce((prev,item) => {
  return (prev + item * item);
}, 0);
console.log("sumOfSquare: " + sumOfSquare);

// output
sumOfSquare: 54

```
此次调用reduce，传了一个初始值0，也就是初始化简结果是0。因为我们最终算的是一个和，初始值0并不影响最终的和运算。

###### reduce方法实现
```JavaScript

if (!Array.prototype.reduce2) {
  Array.prototype.reduce2 = function (func, initial) {
    console.log("arguments: " + arguments.length);
    const len = this.length;
    let i = 0;
    let prev;
    if (arguments.length >= 2) {
      prev  = initial;
    } else {
      // find the first non-empty item

      // if (len == 0) {
      //   // this array is empty.
      //   throw TypeError("Reduce of empty array with no initial value");
      // }

      while (i < len) {
        if (i in this) {
          prev  = this[i];
          break;
        } else {
          ++i;
        }
      }
      if (i == len) {
        // this array is empty.
        throw TypeError("Reduce of empty array with no initial value");
      }
    }

    while(i < len) {
      if ( i in this) {
        prev = func.call(null, prev, this[i], i, this)
      }
      ++i;
    }
    
    return prev;
  };
}

```

调用自定义reduce的例子：
```JavaScript

var srcArray = [2,,3,4,5];
var sumOfSquare = srcArray.reduce2((prev,item,index,arr) => {
  console.log(prev + " " + item + " " + index + ";    " + arr);
  return prev + item * item;
}, 0);

console.log(sumOfSquare);

// output 
arguments: 2
0 2 0;    2,,3,4,5
4 3 2;    2,,3,4,5
13 4 3;    2,,3,4,5
29 5 4;    2,,3,4,5
54

```

## forEach (ES5)
forEach的参数是一个函数，它将为每个元素调用一次该参数函数（除了**空缺元素**，空缺元素可以看做是占用索引位置但是本身又不存在的元素，与值为undefined的元素是有区别的）。forEach会使用三个参数调用参数函数：数组元素、元素索引以及数组本身。参数函数可以根据需要只接收前n个参数（n <= 3），**超过3个的形参将被赋值为undefined**。比如：
```JavaScript

var arr = [1,,undefined,null,false,"hello"];

arr.forEach((item,index) => {
  console.log(item + " " + index);
});

// output
1 0
undefined 2
null 3
false 4
hello 5

```
上面的代码，只关心元素值和元素索引，所以参数函数只定义了两个形参，接收forEach调用参数函数时的前两个参数。

forEach与for-in遍历是有区别的：for-in遍历对象（当然包括数组）属性时，不确定是按顺序的。ES标准并没有规定for-in枚举对象、数组属性时按照顺序来。但是ES5的forEach一定是按照**索引顺序**遍历数组的。

