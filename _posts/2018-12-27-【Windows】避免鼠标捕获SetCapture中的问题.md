---
layout:         page
title:         【Windows】避免鼠标捕获SetCapture中的问题
subtitle:       
date:           2018-12-27
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 鼠标捕获的解决方案是来自一篇英文博客，原文地址：[avoiding-trouble-with-mouse-capture](http://www.drdobbs.com/avoiding-trouble-with-mouse-capture/184416474)   。感谢Chris Branch。

#### 为什么需要捕获鼠标？
做Windows上的客户端开发，不可避免地要处理各种消息，其中包括鼠标相关的消息。为了让一个窗口客户区可以被鼠标拖动，我在 **WM_LBUTTONDOWN** 消息处理时记录一个状态mouse_left_hould=true。在 **WM_MOUSEMOVE** 消息中，判断状态mouse_left_hould为true，则移动窗口到光标当前位置。在 **WM_LBUTTONUP** 消息中取消对鼠标的捕获，并把状态mouse_left_hould标记为false。

一切看起来似乎非常美好。

**但是实际问题总比预想的复杂。** 实际情况下，就算我们捕获了鼠标消息，我们依然有可能收不到 **WM_LBUTTONUP** 消息。比如说，有些版本的Windows系统可能会取消我们对鼠标消息的捕获，在这样的情况下，我们的窗口可能无法再收到 **WM_LBUTTONUP** 消息，因为我们的状态 mouse_left_hould 已经不对了。这个时候，这个窗口就陷入了“dragging”模式，尽管我们并没有按住鼠标左键去拖动它。除了与系统有关，其他还有一些情况下，我们也会被取消对鼠标的捕获，比如：按下Alt+Tab键切换应用程序；按下Win键或者其他组合键弹出开始菜单；或者一个后面的进程调用了SetForegroundWindow()（最后这个我没验证过）。

#### 解决方案
有几个消息能帮助我们解决这个问题。当系统“偷”我们对鼠标的捕获之前，我们的窗口会收到一个 **WM_CANCELMODE** 消息。默认情况下，DefWindowProc() 收到一个 **WM_CANCELMODE** 消息时会调用 ReleaseCapture()。因此，我们可以把这个消息交给DefWindowProc()处理，它会调用ReleaseCapture()。

因此，**一个比较合理的解决方案**是：接收**WM_CANCELMODE**消息并处理，做一些清理工作，比如我这里是标记状态mouse_left_hould=false。然后再把这个消息交给DefWindowProc()处理。

实际上还有一个消息也能解决这个问题：**WM_CAPTURECHANGED** 。不过这个消息与Windows版本有关，另外，经过测试验证，**WM_CANCELMODE** 消息能解决我的问题，我本人采用的方案是处理 **WM_CANCELMODE** 消息。
