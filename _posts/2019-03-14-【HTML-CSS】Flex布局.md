---
layout:         page
title:         【HTML-CSS】Flex布局
subtitle:       
date:           2019-03-14
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 看了阮一峰老师的Flex布局教程，就整理了一下，才有此文。这里基本就是把我实际用到的一些东西摘出来，写成更简短的一篇文章，以作复习之用。另外，为了节约时间，这里的图片也取自阮一峰老师的博客。再次致谢！

#### 什么是Flex布局
**Flex**(**Flexible Box**)，意思是"**弹性布局**"。用来为盒模型提供更加灵活的布局方式。任何一个容器、行内元素，都可以使用Flex布局。Webkit内核的浏览器，使用Flex布局时必须加上-webkit前缀：
```JavaScript

.tag-div{
  display: -webkit-flex; /* Safari */
  display: flex;
}
```
注意，设为Flex布局以后，子元素的float、clear和vertical-align属性将**失效**。

#### Flex的概念
我们把采用Flex布局的元素，称为Flex容器。Flex容器的所有子元素，即Flex容器成员，称为Flex项目（flex item）。Flex容器布局的示意图如下：
![flex layout](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/html_css/flex_layout.png)

Flex容器默认存在两根轴：水平的主轴（**main axis**）和垂直的交叉轴（**cross axis**）。主轴的开始位置（与边框的交叉点）叫做“**main start**”，结束位置叫做“**main end**”；同样，交叉轴也有两个位置“**cross start**”和“**cross end**”。Flex容器内的项目默认沿主轴排列。单个项目占据的主轴空间叫做“**main size**”，占据的交叉轴空间叫做“**cross size**”。

#### Flex布局中常用的属性
##### 作用于Flex容器上的属性
1. flex-direction
2. flex-wrap
3. flex-flow
4. justify-content
5. align-items

类似align-content等作用于多根轴线的属性，不多用，这里不提。

###### flex-direction
flex-direction属性决定主轴的方向（即Flex容器内项目的排列方向）。
```JavaScript

.tag-div{
  flex-direction: row | row-reverse | column | column-reverse;
}

```
它们的效果如下图：
![flex direction](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/html_css/flex_direction.png)

###### flex-wrap
默认情况下，Flex容器内的项目都排在一条线（即"轴线"）上。flex-wrap属性定义：当一条轴线排不下时，如何换行。
```JavaScript

.tag-div{
  flex-wrap: nowrap | wrap | wrap-reverse;
}

```
它们的效果如下图：
![flex wrap](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/html_css/flex_wrap.png)

###### flex-flow
flex-flow属性组合了flex-direction和flex-wrap，默认值为row nowrap。

###### justify-content
justify-content属性定义了项目在**主轴(main axis)上的对齐方式**。
```JavaScript

.tag-div{
  justify-content: flex-start | flex-end | center | space-between | space-around;
}

```
它们的效果分别如下图：
![justify-content](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/html_css/justify_content.png)

**特别说明**：
1. 对于space-between，Flex容器内的项目会两端对齐，项目之间的间隔都相等。
2. 对于space-around，Flex容器内的项目两侧的间隔相等。也因为这个特性，这些项目之间的间隔是两端的项目与边框的间隔的两倍。

###### align-items
align-items属性定义Flex容器内的项目在交叉轴（cross axis）上的对齐方式。
```JavaScript

.tag-div{
  align-items: flex-start | flex-end | center | baseline | stretch;
}

```
它们的效果分别如下图：
![align-items](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/html_css/align_items.png)

**特别说明**：
1. 对于baseline，Flex容器内的项目的第一行文字的基线会对齐.
2. 对于stretch（这也是默认值），如果项目未设置高度或设为auto，则它的高度将占满整个Flex容器的高度。

#### 作用于Flex项目上的属性
Flex项目也可以设置一些属性，比如order(定义项目排列顺序)等，不过并不属于布局，这篇文章只说Flex布局，就不提Flex项目的属性了，需要用的时候查文档即可。

#### 参考
[Flex 布局教程（阮一峰的网络日志）](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html?)