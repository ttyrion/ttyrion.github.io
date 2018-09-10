---
layout:     page
title:      【Linux】Debug with VS Code
subtitle:   Linux系统上用VS Code开发和调试
date:       2018-09-09
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 前言

>code.visualstudio.com上有比较详细的介绍在Linux系统中用VS Code开发和调试的资料，这里记录我自己配置调试环境的过程。原文链接在[这里](https://code.visualstudio.com/docs/languages/cpp)

## VS Code环境
安装 VS Code 和 Microsoft C/C++ extension 的过程很简单，官网有介绍，就不再介绍了。这里只是提一下，官网上可下载的Linux系统的 VS Code 有.deb, .rpm, .tar.gz 等格式，个人建议下载 .tar.gz 格式，解压即可用。
下载地址在[这里](https://code.visualstudio.com/#alt-downloads).  

## 开始调试
### 一、 Open Code Folder
VS Code基于文件夹管理项目，不像VS Studio那样通过 .sln 管理解决方案, 通过 .vcxproj 管理项目。因此，第一步我们打开代码文件夹，如下图：  
![打开文件夹](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/linux/vscode/start.png)  

### 二、Build Code
VS Code 通过一个 tasks.json 文件来 build 项目代码。创建 tasks.json 文件的步骤如下：  
1. 打开**Command Palette**（快捷方式**Ctrl+Shift+P**）
2. 选择 "Tasks: Configure Tasks..." 这个命令, **Command Palette**会以一个下拉列表的形式给出一些可选择的命令，如果没有需要的命令, 比如 "Tasks: Configure Tasks...", 可以输入"tasks"，
**Command Palette**的下拉列表就会过滤出符合要求命令。之后再点击**Create tasks.json file from templates**，下拉列表中会出现可供选择的一系列任务启动模板。  
3. 选择**Others**，创建一个运行外部命令的Build任务。至此，从 VS Code 左侧浏览区域可以看到，我们的 SAMPLE 目录下面多了一个 ".vscode" 目录，该目录下包含一个 tasks.json 文件。实际上，VS Code 确实在
我们的SAMPLE代码目录中创建了一个名为".vscode"的隐藏目录。
4. 修改默认的 tasks.json 文件。默认的tasks.json文件内容如下：  
```json
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "echo",
            "type": "shell",
            "command": "echo Hello"
        }
    ]
}
```
"label"可以认为是一个task的名字，后面启动这个task的时候就需要指定task的label，因此我们这里修改 "label" 的值为 "Build Sample"。
"command"是这个task要执行的外部命令，我这里直接改成 "make", 因为我的sample项目是通过 make 管理的，**build**过程只需要执行 make Makefile。
改过的tasks.json文件如下：
```json
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build Sample",
            "type": "shell",
            "command": "make"
        }
    ]
}
```
**注意：这里是要调试代码，所以build时要加 "-g" 参数以支持调试，因此，我把原先的Makefile中的 CC = g++，改为 CC = g++ -g .**
Build Code 这一步结果如下图：  
![Build Code](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/linux/vscode/tasks.png)  


### 三、Debugging Code
要调试代码，还需要创建一个 launch.json 文件。这个文件主要是告诉 VS Code，如何启动调试：比如启动什么调试器，调试的进程的程序文件路径，调试前还需要执行什么"task"等等。
1. 点击左侧工具栏中的"Debug"图标（一个圆圈中间包围了一只臭虫，挺形象的一个图标），切换到Debug视图。
2. 点击工具栏右边浏览区域右上角的设置按钮，选择 **C++ (GDB/LLDB)**（使用GDB或者LLDB），这会创建一个 launch.json 文件。这个文件可以配置两种调试方式：**C++ Launch** 或者 **C++ Attach**。
3. 修改 "program"的值为要调试的程序文件路径。
4. 如果想要每次调试之前都Build一下项目，就可以给launch.json文件增加一个 "preLaunchTask" 属性，它的值配置成某个Build任务的名称，例如上面的 tasks.json 中配置的那个
building task 的"label"属性的值"Build Sample"。
修改后的launch.json文件内容如下：
```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/bin/sample",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": true,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "Build Sample"
        }
    ]
}
```

最终的调试界面如图：  
![Debugging Code](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/linux/vscode/debugging.png)  
