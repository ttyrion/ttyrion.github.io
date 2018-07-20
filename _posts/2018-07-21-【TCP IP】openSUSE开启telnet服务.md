---
layout:     page
title:      【TCP IP】openSUSE开启telnet服务
subtitle:   telnet、xinetd
date:       2018-07-21
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---
## install telnet-server and xinetd
openSUSE默认只安装了telnet客户端，而没有安装服务端，因此我们要先安装telnet服务端。
另外，我们可以用xinetd超级守护进程来管理telnet服务。  
sudo zypper in telnet-server  
这个时候可以看到telnet服务程序已经安装：/usr/sbin/in.telnetd。  
但是telnet服务没有启动：  
$ telnet 127.0.0.1  
Trying 127.0.0.1...  
telnet: connect to address 127.0.0.1: Connection refused  
连不上telnet服务，再次查看端口确认：  
netstat -tnl | grep 23  
没有任何输出，表示确实没有telnet服务。  
$ sudo zypper in xinetd  
$ cd /etc/xinetd.d  
$ ls  
chargen      daytime      discard      echo      servers   time-udp  
chargen-udp  daytime-udp  discard-udp  echo-udp  services  time  
进入xinetd配置目录查看，发现没有telnet。  
那么自己新建一个：  
$sudo vi telnet  
输入如下内容:
```javascript
#default: on  
#description: telnet serves telnet sessions  
service telnet
{
	flags = REUSE
	socket_type = stream //tcp
	wait = no //that is , more than one users can access at the same time
	user = root
	server = /usr/sbin/in.telnetd //the process, telnet server
	log_on_failure += USERID
	disable = no //enable telnet server
}
```
注意：写入telnet配置文件的时候，上面的注释是要去掉的。  
接下来重提xinetd：  
$ sudo service xinetd restart  
$ netstat -tnl | grep 23  
tcp6       0      0 :::23                   :::*                    LISTEN  
可见，23端口已经有服务在监听了。  
$ telnet 127.0.0.1  
Trying 127.0.0.1...  
Connected to 127.0.0.1.  
Escape character is '^]'.  

Linux 4.12.14-lp150.11-default (192) (1)  

192 login:   

至此，telnet服务已经启动了。  
