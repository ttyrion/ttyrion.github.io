---
layout:     page
title:      【糗事】安装Win10+openSUSE双系统之后无法找到openSUSE启动项
subtitle:   
date:       2018-09-19
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 前言

> 为了犒劳自己，若日入手一台新笔记本。安装完Win10后接着在保留的未分区的磁盘中安装openSUSE，结果启动项里面看不到openSUSE，只能启动Win10。奇怪的是以U盘openSUSE启动后选择从硬盘启动系统时，却能看到openSUSE启动项。昨晚整到半夜无果，今日下班回家继续折腾。。。

先看几个概念，这是我最后能找到问题的基础。为了不至于描述错误（本人这方面的知识不够科普），以下几个概念的解释主要来自[openSUSE 官网](https://zh.opensuse.org/openSUSE:UEFI)。

## UEFI
UEFI（Unified Extensible Firmware Interface, 统一可扩展固件接口）是一个新的工业标准，指定了一台计算机必须在预引导环境提供的不同接口。UEFI 将在开机后和操作系统完全加载时控制计算机。UEFI 也负责提供介于计算机提供的资源和操作系统间的接口。换句话说，在此 UEFI 用于替代和扩展旧式的 BIOS 固件。

UEFI 的一个核心点就是它是可扩展的。UEFI 有一个内部的虚拟机，该虚拟机独立于它运行的架构。该标准接受为此虚拟机编译的特定二进制文件（EFI 执行文件），并在此环境中执行。这些执行文件可以是设备驱动、应用程序或 UEFI 标准的扩展。**UEFI，在某种意义上说，像是计算机启动时运行的一个小型操作系统，该操作系统的主要任务是查找并加载另一个主力操作系统。**

## GPT
UEFI 改动的不止有固件、syscall 和接口。它也提议了一种新的硬盘分区风格。GUID 分区表 (GPT) 在此将替代老式的主引导记录 (MBR) 方案。MBR 有一些重要的限制，比如主分区和逻辑分区的数量，和这些分区的大小等。GPT 解决了这些问题，现在我们可以使用任意数量的分区 (in sets of 128)，可寻址的磁盘空间也达到了 2^64 比特 。因此，如果我们有大硬盘的话，这种分区方式正式我们所要的。另一个核心区憋局势 GPT 使用一个独一无二的 UUID 数引用每个分区，避免了分区标识符间的撞车。

## EFI 系统分区
EFI 系统分区（**ESP**）是 UEFI 期望找到可用于引导全部安装在设备上的操作系统的 EFI 程序的分区。同时，EFI 也将在这里查找一些引导时所用的设备驱动，以及其它需要在操作系统引导前运行的工具。此分区使用 FAT 文件系统，可通过 YaST2 在全新安装时创建出来，双重引导时也可使用别的系统创建好的。这意味着若我们在计算机上已经安装了一个 Windows，YaST2 将能检测到 EFI 分区，并将会把用于加载 openSUSE 的新 EFI 引导加载器放在里面，并且在 /boot/efi 挂载点下挂载该分区。

## 发现问题
这个问题已经持续到第二天，无奈开始从头排查：从磁盘分区开始看。从Win10“磁盘管理”工具突然看到了两个“EFI 系统分区”。至此恍然大悟，肯定是安装openSUSE时又创建了新的ESP，因此系统启动项中看不到openSUSE，但是开机启动设置（BIOS设置）中却能看到openSUSE。于是问题来了，怎么确认这两个ESP哪个是Win10创建的，哪个是openSUSE创建的？肯定要查看ESP分区中的文件。以下是过程记录:

以管理员权限启动CMD，执行下面的命令：

```
C:\WINDOWS\system32>diskpart

Microsoft DiskPart 版本 10.0.17134.1

Copyright (C) Microsoft Corporation.
在计算机上: DESKTOP-FSKGDR9

DISKPART> list disk

  磁盘 ###  状态           大小     可用     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  磁盘 0    联机              447 GB   247 GB        *

DISKPART> select disk 0

磁盘 0 现在是所选磁盘。

DISKPART> list partition

  分区 ###       类型              大小     偏移量
  -------------  ----------------  -------  -------
  分区      1    恢复                 450 MB  1024 KB
  分区      2    系统                 100 MB   451 MB
  分区      3    保留                  16 MB   551 MB
  分区      6    系统                 500 MB   567 MB
  分区      4    主要                 198 GB   248 GB
  分区      5    恢复                 832 MB   446 GB

DISKPART>

```
从上面列出的磁盘0的分区中，可以看到确实有两个“系统”分区，这就是ESP。接下来就是查看分区数据了。

```
DISKPART> select partition 2

分区 2 现在是所选分区。

DISKPART> assign letter=M:

DiskPart 成功地分配了驱动器号或装载点。

DISKPART> select partition 6

分区 6 现在是所选分区。

DISKPART> assign letter=N:

DiskPart 成功地分配了驱动器号或装载点。

DISKPART>

```
上面的命令分别给两个ESP分区挂载到盘符 M 和 N。接下来就可以查看分区数据。

```
DISKPART> exit

退出 DiskPart...

C:\WINDOWS\system32>cd /d M:\

M:\>dir
 驱动器 M 中的卷没有标签。
 卷的序列号是 043A-A68F

 M:\ 的目录

2018/06/29  23:03    <DIR>          EFI
               0 个文件              0 字节
               1 个目录     74,257,408 可用字节

M:\>dir EFI
 驱动器 M 中的卷没有标签。
 卷的序列号是 043A-A68F

 M:\EFI 的目录

2018/06/29  23:03    <DIR>          .
2018/06/29  23:03    <DIR>          ..
2018/06/29  23:03    <DIR>          Microsoft
2018/06/29  23:07    <DIR>          Boot
               0 个文件              0 字节
               4 个目录     74,257,408 可用字节

M:\>
```
从上面的结果可以看到，M盘里面有个EFI目录，这里面有Microsoft的启动设置。

```
M:\>cd /d N:\

N:\>dir
 驱动器 N 中的卷没有标签。
 卷的序列号是 23E1-E30B

 N:\ 的目录

2018/09/19  00:44    <DIR>          EFI
               0 个文件              0 字节
               1 个目录    518,766,592 可用字节

N:\>dir EFI
 驱动器 N 中的卷没有标签。
 卷的序列号是 23E1-E30B

 N:\EFI 的目录

2018/09/19  00:44    <DIR>          .
2018/09/19  00:44    <DIR>          ..
2018/09/19  00:44    <DIR>          boot
2018/09/19  00:44    <DIR>          opensuse
               0 个文件              0 字节
               4 个目录    518,766,592 可用字节

N:\>
```
这里可以看到，N盘里面的EFI是openSUSE创建的启动设置。
至此就可以得出结论：2号分区，大小为100M，是安装Win10时创建的。而6号分区，大小为500M，是openSUSE安装过程中创建的。

应该有办法合并这两个分区解决问题，不过我这里openSUSE系统被我倒腾得启动不了了，遂删除了6号分区，重新安装openSUSE。








