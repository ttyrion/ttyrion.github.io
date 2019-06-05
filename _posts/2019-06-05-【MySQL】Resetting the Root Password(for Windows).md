---
layout:         page
title:         【MySQL】Resetting the Root Password(for Windows)
subtitle:       
date:           2019-06-05
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

MySQL官网就有关于怎么重置root密码的文档：[Chapter 4 Resetting the Root Password: Windows Systems](https://dev.mysql.com/doc/mysql-windows-excerpt/5.7/en/resetting-permissions-windows.html)。这里只是记录一下本人实际操作的过程。

#### 停止MySQL服务
1. 如果是通过系统服务启动MySQL（而不是通过mysqld进程），则现在系统服务（win10搜索“服务”）中停止MySQL服务。
2. 如果是通过mysqld进程启动的MySQL服务，终止进程。

#### 创建重置密码的文本
创建一个文本文件（这里假设文件路径是C:\mysql-init.txt），添加密码赋值语句：
```javascript

ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourNewPassword';

```
MySQL 5.7.5 及以前的版本这条语句有所不同，具体可看官方文档。

#### 命令行启动mysqld进程
```javascript

mysqld.exe --init-file=C:\\mysql-init.txt --console

```

注意，上面加了参数--console，让mysqld把日志输出到命令行，而不是写入日志文件。

没有问题的话，至此密码已经修改成功并且启动了MySQL服务。现在可以删除上面那个文本文件C:\mysql-init.txt。