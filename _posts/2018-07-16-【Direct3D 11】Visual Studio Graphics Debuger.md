---
layout:     page
title:      【Direct3D 11】Visual Studio Graphics Debugger 
subtitle:   VS着色器调试
date:       2018-07-16
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

# 启动VS图形调试器
要启动VS图形调试器：  
选择菜单：“调试”->“图形”->“启动图形调试”。  
![图形调试器](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/direct3d/debug/launch-debugger.png)  

图形（着色器）调试和传统的调试有些不同。  
因为着色器代码是基于顶点和像素执行的，所以我们需要先指明我们要调试哪个像素。为了选择一个像素，我们先要抓取一帧数据。  
抓一帧数据可以点击上图左下角的“捕获帧”按钮，或者按下Print Screen键。抓到的帧如下图：  

![帧](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/direct3d/debug/frame.png)  

这时可以在VS中看到被抓到的帧数据列表。双击其中某一帧，可以启动VS图形分析器：  

![图形分析器](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/direct3d/debug/graphics_analyzer.png)  

鼠标在帧中选择某一个像素，可以看到选中的像素会被红色十字光标标记。右侧是该像素的历史记录，其中也记录了该像素在顶点着色器  
和像素着色器的处理过程。旁边的绿色小三角按钮，便是启动着色器调试的按钮。调试器界面如下图：  
 
![图形分析器](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/direct3d/debug/debug.png)  

可能是因为着色器代码优化，有些代码行不能下断点。图形分析器左下角可以看到着色器局部变量和常量缓冲。  


  
  
  
  

