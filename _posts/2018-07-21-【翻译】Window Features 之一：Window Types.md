---
layout:     page
title:      【Windows】Window Features 之一：Window Types(翻译)
subtitle:   MSDN上的一篇介绍窗口特性的“总纲”中的Window Types
date:       2018-07-29
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

# 前言

>这篇文章是翻译一篇MSDN上介绍窗口特性的文章，原文地址：https://docs.microsoft.com/en-us/windows/desktop/winmsg/window-features#window-relationships

此篇翻译包括以下几个描述 Window types 的主题：
1. **Overlapped Windows**
2. **Pop-up Windows**
3. **Child Windows**
4. **Layered Windows**
5. **Message-Only Windows**

### Overlapped Windows
一个 overlapped 窗口是一个 top-level 窗口，它包含标题栏、边框、客户区， 用作应用程序的主窗口。overlapped 窗口也可以拥有菜单、最大化最小化按钮、滚动条。 用作主窗口的overlapped 窗口通常拥有上述所有组件。
应用程序可以通过以下方式创建一个 overlapped 窗口：调用 CreateWindowEx 时指定 WS_OVERLAPPED 或者 WS_OVERLAPPEDWINDOW 风格。如果指定了 WS_OVERLAPPED，这个窗口会包含一个标题栏和边框。如果指定了 WS_OVERLAPPEDWINDOW，
这个窗口会包含一个标题栏、可调整大小的边框、菜单、最大化最小化按钮。  

### Pop-up Windows
pop-up 窗口是一种特殊类型的 overlapped 窗口，用于 dialog boxes, message boxes, 以及其他可以出现在应用程序主窗口之外的临时窗口。 标题栏对于 pop-up 窗口来说是可选的，否则，pop-up 窗口就和 WS_OVERLAPPED 风格
的 overlapped 窗口一样。
调用 CreateWindowEx 并制定 WS_POPUP，即可创建一个 pop-up 窗口。 还可以同时指定 WS_CAPTION， 以便让窗口包含一个标题栏。指定 WS_POPUPWINDOW 风格可以创建一个包含边框和菜单的 pop-up 窗口。必须同时指定 WS_POPUPWINDOW 和 WS_CAPTION 才能让窗口菜单可见。
```
鉴于pop-up窗口的特性，使用duilib界面库的应用程序通常都用 Pop-up 窗口作为主窗口，创建窗口的时候指定  WS_POPUP ，但不需要指定 WS_POPUPWINDOW 或 WS_CAPTION。因为我们通常用自绘的菜单，以及自定义的标题栏（区域）。
```

### Child Windows
子窗口拥有 WS_CHILD 风格，并且被限制在父窗口的客户区中。应用程序通常用子窗口来把父窗口的客户区划分成多个功能区。 调用 CreateWindowEx 时指定 WS_CHILD 可以创建一个子窗口。  

一个子窗口必须拥有一个父窗口。父窗口可以是 overlapped 窗口, pop-up 窗口, 甚至可以是另一个子窗口。父窗口也在调用 CreateWindowEx 时指定。如果调用 CreateWindowEx 时指定了 WS_CHILD 窗口风格，但是没有指定父窗口，
那么系统不会窗口该窗口。  

子窗口有一个客户区，但是没有其他特性，除非明确地请求某些特性。应用程序可以为子窗口请求一个标题栏、窗口菜单、最大化最小化按钮、边框、滚动条， 但是子窗口不能有菜单。
如果没有为子窗口指定边框风格，系统会创建一个无边框的窗口。应用程序可以用无边框的子窗口来划分父窗口的客户区，并且这些分界线对用户是不可见的。  

