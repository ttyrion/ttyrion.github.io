---
layout:         page
title:          【Linux】后端定时任务：crontab与flock的结合
subtitle:       
date:           2019-07-08
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

后端有些重复性的工作，例如数据刷新等，经常使用定时任务来完成。这里简单介绍通过crontab以及flock完成此项工作。

### cron
cron是Linux的一个负责调度任务的daemon程序，查看系统进程就能看到一个名为crond的进程。cron可根据指定的时间间隔执行各种任务。这些任务就称为“cron job”，通常都用于系统维护等工作。我们可以通过cron每隔一分钟、一小时、一天...或者它们的组合，来调度我们的cron job。

#### crontab file
crontab(cron table)文件是一个指定了如何调度cron jobs的文本文件。有两种类型的crontab文件：系统范围以及特定用户的。
1. 用户的crontab文件根据用户名存储，其存储路径因系统而异。在类似CentOS等基于Red Hat的系统中，crontab文件存储在/var/spool/cron目录中；在Ubuntu等系统中，crontab文件存储在/var/spool/cron/crontabs目录中。例如，以本人的CentOS虚拟机为例，系统中存在一个/var/spool/cron/testuser文件，文件内容就是被调度的几个cron jobs。
1. /etc/crontab文件以及/etc/cron.d目录下的文件，都是系统的crontab文件。

你可以直接编辑crontab文件（如上述/var/spool/cron/testuser文件），但仍建议你使用**crontab命令**来。

#### crontab语法以及运算符
```javascript
* * * * * command(s)
- - - - -
| | | | |
| | | | ----- Day of week (0 - 7) (Sunday=0 or 7)
| | | ------- Month (1 - 12)
| | --------- Day of month (1 - 31)
| ----------- Hour (0 - 23)
------------- Minute (0 - 59)
}

```
开头的五个字段可以包含一个或多个值。
1. \* ： 星号运算符表示任何值或总是（any value or always）。如果在Hour字段中包含一个星号，表示这个任务每个小时都会执行。
1. ,  ： 逗号允许我们指定一系列值来重复调度任务。比如，在Hour字段中指定1,3,5 ，这个任务就会在1点、3点、5点被执行。
1. \- ： 连字符允许指定一个范围的值。比如，在 Day of week 字段中指定1-5，这个任务就会在每隔工作日被执行。
1. /  ： 反斜线允许我们指定每隔一定的时间间隔来重复调度任务。比如，在Hour字段中指定*/4，每隔四小时，这个任务都会被执行。这与指定0,4,8,12,16,20效果相同。这里的星号就表示“任何值”。这里也可以不指定*,而是使用一个范围值，如1-30/10，这和指定1,11,21相同。

#### Predefined Macros
有些特殊的cron调度宏，可被用于指定一些通用的时间间隔。
1. **@weekly** 每周日的午夜执行指定的任务。等同于0 0 * * 0。
1. **@daily** 每天午夜执行指定的任务。等同于0 0 * * *。
1. **@hourly** 每小时开始时执行指定的任务。等同于0 * * * *。
1. **@reboot** 在系统启动时执行指定的任务。

#### Linux Crontab Command
crontab命令允许我们安装或者打开crontab文件，我们可以使用crontab命令来查看，添加，移除，修改cron jobs：
1. **crontab -e** 编辑crontab文件，如不存在则创建一个。
1. **crontab -l** 查看crontab文件的内容。
1. **crontab -r** 移除（remove）用户当前的crontab文件。
1. **crontab -u \<username\>** 编辑其他用户的crontab文件（需要超级用户权限）。

#### Crontab Restrictions
系统管理员可以通过/etc/cron.deny文件以及/etc/cron.allow文件来控制哪些用户拥有执行crontab命令的权限。这两个文件内容是一个用户名列表，每个用户名占一行。

默认情况下，系统中只有一个/etc/cron.deny文件，并且其内容为空，藐视所有的用户可以执行crontab命令。如果管理员想禁止某用户使用crontab命令，则将其用户名添加到此文件中即可。

如果系统中存在/etc/cron.allow文件，则只有这个文件中列出的用户才能使用crontab命令。

如果这两个文件都不存在，则只有管理员可以执行crontab。

#### Cron Jobs Examples
每个工作日（周一至周五）的下午三点执行任务:
```javascript
0 15 * * 1-5 command
```

每隔5分钟执行脚本并且把标准输出重定向到dev null，只有标准错误会被发送至指定的email地址：
```javascript
MAILTO=email@example.com
*/5 * * * * /path/to/script.sh > /dev/null
```

设置自定义的crontab环境变量：
```javascript
HOME=/opt
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
SHELL=/bin/zsh
MAILTO=email@example.com
```

### flock
flock命令可以在shell脚本或命令行环境中管理flock（api）锁。
```javascript
flock [-sxon] [-w timeout] lockfile [-c] command...
flock [-sxon] [-w timeout] lockdir [-c] command...

flock [-sxun] [-w timeout] fd
```
第一、二中形式中，flock锁住指定的文件或者目录，如果它们不存在，flock会创建它们（假设有合适的权限）。

默认情况下，如果不能立即获得文件锁，flock会等待，直到文件锁可用。

#### flock选项
**-x, -e, --exclusive**
获取互斥锁，有时也叫作写锁。这是默认选项。

**-n, --nb, --nonblock**
如果不能立即获取文件锁，则失败（进程退出状态码为1）而不是等待。

**-w, --wait, --timeout seconds**
如果不能在指定的seconds内获取文件锁，则失败（进程退出状态码为1）而不是等待。

**-c, --command command**
传递一个简单的命令给shell。

#### flock与crontab结合
例如，这里有一个简单的脚本main.py：

```javascript
// main.py

import sys
import time
import datetime

for i in range(30):
    if len(sys.argv) >= 2:
        print('{0:%H:%M:%S}'.format(datetime.datetime.now()) + ":" sys.argv[1])
        time.sleep(10)
```

且有如下的定时任务文件test.cron：
```javascript
// test.cron

*/2 * * * * flock -xn /tmp/1.lock -c "python /home/testuser/main.py 1 >> /home/testuser/test.log"
*/2 * * * * flock -xn /tmp/1.lock -c "python /home/testuser/main.py 2 >> /home/testuser/test.log"
*/2 * * * * flock -xn /tmp/3.lock -c "python /home/testuser/main.py 3 >> /home/testuser/test.log"
```

通过crontab test.cron来安装这几个cron jobs，并观察日志文件，可看到每隔两分钟，任务3都起来了，而任务1和2不能同时起来。

```javascript
// test.log

Task3 and Task1 run at the same time:
15:32:01    3     15:32:01    1
15:32:11    3     15:32:11    1
15:32:21    3     15:32:21    1
15:32:31    3     15:32:31    1
15:32:41    3     15:32:41    1
15:32:51    3     15:32:51    1
15:33:01    3     15:33:01    1
15:33:11    3     15:33:11    1
15:33:21    3     15:33:21    1
15:33:31    3     15:33:31    1
15:33:41    3     15:33:41    1
15:33:51    3     15:33:51    1
15:34:01    3     15:34:01    1
15:34:12    3     15:34:12    1
15:34:22    3     15:34:22    1
15:34:32    3     15:34:32    1
15:34:42    3     15:34:42    1
15:34:52    3     15:34:52    1
15:35:02    3     15:35:02    1
15:35:12    3     15:35:12    1
15:35:22    3     15:35:22    1
15:35:32    3     15:35:32    1
15:35:42    3     15:35:42    1
15:35:52    3     15:35:52    1
15:36:02    3     15:36:02    1
15:36:12    3     15:36:12    1
15:36:22    3     15:36:22    1
15:36:32    3     15:36:32    1
15:36:42    3     15:36:42    1
15:36:52    3     15:36:52    1

Next, Task3 and Task2 run at the same time:
15:38:01    2     15:38:01    3
15:38:11    2     15:38:11    3
15:38:21    2     15:38:21    3
15:38:31    2     15:38:31    3
15:38:41    2     15:38:41    3
15:38:51    2     15:38:51    3
15:39:01    2     15:39:01    3
15:39:11    2     15:39:11    3
15:39:21    2     15:39:21    3
15:39:31    2     15:39:31    3
15:39:41    2     15:39:41    3
15:39:51    2     15:39:51    3
15:40:01    2     15:40:01    3
15:40:11    2     15:40:11    3
15:40:22    2     15:40:22    3
15:40:32    2     15:40:32    3
15:40:42    2     15:40:42    3
15:40:52    2     15:40:52    3
15:41:02    2     15:41:02    3
15:41:12    2     15:41:12    3
15:41:22    2     15:41:22    3
15:41:32    2     15:41:32    3
15:41:42    2     15:41:42    3
15:41:52    2     15:41:52    3
15:42:02    2     15:42:02    3
15:42:12    2     15:42:12    3
15:42:22    2     15:42:22    3
15:42:32    2     15:42:32    3
15:42:42    2     15:42:42    3
15:42:52    2     15:42:52    3

Next, Task3 and Task1 run at the same time:
15:44:01    1      15:44:01    3
15:44:11    1      15:44:11    3
15:44:21    1      15:44:21    3
15:44:31    1      15:44:31    3
15:44:41    1      15:44:41    3
15:44:51    1      15:44:51    3
15:45:01    1      15:45:01    3
15:45:11    1      15:45:11    3
15:45:21    1      15:45:21    3
15:45:31    1      15:45:31    3
15:45:41    1      15:45:41    3
15:45:51    1      15:45:51    3
15:46:01    1      15:46:01    3
15:46:11    1      15:46:11    3
15:46:21    1      15:46:21    3
15:46:31    1      15:46:31    3
15:46:41    1      15:46:41    3
15:46:51    1      15:46:51    3
15:47:01    1      15:47:01    3
15:47:11    1      15:47:11    3
15:47:21    1      15:47:21    3
15:47:31    1      15:47:31    3
15:47:41    1      15:47:41    3
15:47:51    1      15:47:51    3
15:48:01    1      15:48:01    3
15:48:11    1      15:48:11    3
15:48:21    1      15:48:21    3
15:48:31    1      15:48:31    3
15:48:41    1      15:48:41    3
15:48:51    1      15:48:51    3
```