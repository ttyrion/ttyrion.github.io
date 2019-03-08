---
layout:         page
title:         【Mysql】设置远程连接Mysql数据库
subtitle:       
date:           2019-03-09
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

#### Remote Access to Mysql 
我这里都是安装完Mysql的初始环境，没做任何设置。

```JavaScript
1. 远程访问Mysql，首先要知道Mysql占用的端口：
mysql> show global variables like 'port';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3306  |
+---------------+-------+
1 row in set (0.02 sec)
可见Mysql占用了3306端口（默认端口）。

此时使用IP地址以及用户名、密码连接Mysql依然失败了，原因是没有开启远程访问权限。

2. 查看Mysql访问权限控制：
mysql> use mysql;
Database changed
mysql> select host,user from user;
+-----------+---------------+
| host      | user          |
+-----------+---------------+
| localhost | mysql.session |
| localhost | mysql.sys     |
| localhost | root          |
+-----------+---------------+
3 rows in set (0.00 sec)
从这里可以看到目前只能从localhost登录Mysql进行操作。

将host字段的值改为‘%’就表示在任何客户端机器上能以root（user字段）用户登录到mysql服务器，因此开发时设置host为‘%’会非常有用。
mysql> update user set host='%' where user='root';
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

除了设置host，还要设置访问权限，这里设置为all privileges：
mysql> grant all privileges on *.* to root@'%' identified by 'root123';
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements
上面的错误表示我们的密码格式错误，不符合Mysql的密码策略。
那要设置一个符合Mysql密码策略的密码，就得先知道Mysql的密码有什么要求：
mysql> show variables like 'validate_password%';
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password_check_user_name    | OFF    |
| validate_password_dictionary_file    |        |
| validate_password_length             | 8      |
| validate_password_mixed_case_count   | 1      |
| validate_password_number_count       | 1      |
| validate_password_policy             | MEDIUM |
| validate_password_special_char_count | 1      |
+--------------------------------------+--------+
7 rows in set (0.01 sec)

从上面可以看到，密码要求混合大小写，长度为8，一个数字等等。重新设置权限以及外部登录的密码：
mysql> grant all privileges on *.* to root@'%' identified by 'Amtin@s1';
Query OK, 0 rows affected, 1 warning (0.01 sec)

至此，Mysql的设置完成了。
可是远程连接数据库的时候，依然失败，提示错误：
Lost connection to MySQL server at 'reading initial communication packet', system error: 0.

原因是服务器防火墙限制了对端口的访问。

可以在服务器查看一下防火墙设置（用firewalld操作防火墙非常方便）：
[root@localhost ~]# firewall-cmd --zone=public --list-all （查看默认的public zone）
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens33
  sources:
  services: ssh dhcpv6-client http
  ports: 80/tcp
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
从上面可以看到当前只开放了80端口。
那么我们要再开放3306端口（--permanent表示永久性的，重启有效）：
[root@localhost ~]# firewall-cmd --permanent --zone=public --add-port=3306/tcp
success
启用新的防火墙设置以后再次查看：
[root@localhost ~]# firewall-cmd --reload
success
[root@localhost ~]# firewall-cmd --zone=public --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens33
  sources:
  services: ssh dhcpv6-client
  ports: 80/tcp 3306/tcp
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
此时可以看到，3306端口已经开放了。

```




## Mysql rpm bundle for linux
[download link](https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.25-1.el7.x86_64.rpm-bundle.tar)