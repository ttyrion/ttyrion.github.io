---
layout:     page
title:      【Direct3D 11】文字渲染之篇三：Surface Sharing
subtitle:   Direct2D, DirectWrite, Direct3D 10.1 and Direct3D 11
date:       2018-08-19
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---
## 前言

>[这里](https://www.braynzarsoft.net/viewtutorial/q16390-14-simple-font) 是一篇英文的介绍用共享Surface在Direct3D 11中渲染文字的帖子,我这里也参考了这篇文章。文中把流程讲的比较仔细，不过我认为逻辑上有点乱，且Alpha混合的代码有问题。作为参考，还是很不错的。

## Direct3D 11 中的文字渲染问题
