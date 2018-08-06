---
layout:     page
title:      【TCP IP】CDN网络和DNS的一点浅析
subtitle:   CDN怎么利用DNS服务来分发
date:       2018-08-05
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## DNS的CNAME RR：域名重定向
CNAME被称为“规范名字”，可以认为是一个别名或者域名重定向。
比如：  
C:\Users\santiago>**nslookup**  
默认服务器:  UnKnown  
Address:  ...  
\> **set type=a**  
\> **www.iqiyi.com**  
服务器:  UnKnown  
Address:  ...  
非权威应答:  
名称:    static.dns.iqiyi.com  
Addresses:  ...  
Aliases:  www.iqiyi.com  
\> **vodguide.pps.iqiyi.com**  
服务器:  UnKnown  
Address:  ...  
非权威应答:  
名称:    static.dns.iqiyi.com  
Addresses:  ...  
Aliases:  vodguide.pps.iqiyi.com  

上面的例子可以看出，www.iqiyi.com和vodguide.pps.iqiyi.com都是static.dns.iqiyi.com的一个别名。
就是说，用户访问www.iqiyi.com或者vodguide.pps.iqiyi.com，实际上最后都访问的是static.dns.iqiyi.com。
而访问static.dns.iqiyi.com，就会走爱奇艺的CDN网络。

## CDN网络
CDN网络架构主要由两大部分: 中心和边缘两部分。  
1. **中心**指CDN网络管理中心和DNS重定向解析中心  
中心部分负责全局负载均衡。设备系统安装在管理中心机房。
2. **边缘**主要指异地节点，CDN分发的载体  
CDN节点主要由缓存服务器和负载均衡等组成。  

CDN网络对域名解析过程是进行调整了的。CDN提供服务的原理可以概括为以下几步：  
1. 域名重定向  
就如上面说到的，www.iqiyi.com和vodguide.pps.iqiyi.com都被重定向到static.dns.iqiyi.com。当用户访问加入CDN服务的网站时，
域名解析请求将最终交给DNS重定向解析中心（因为它要负责全局的负载均衡）进行处理。中心根据某种策略，将当时最接近用户的
节点地址提供给用户。  
2. CDN管理中心要返回用户一个CDN节点地址  
中心根据某种策略，将当时最接近用户的节点地址提供给用户。    
3. 原始域名  
CDN节点还可能要根据原始域名返回给用户相应内容。比如上面的例子，两个域名都被指定到同一个CDN服务。
原始域名可以在http头里面取到。    
4. CDN节点缓存服务器请求源服务器  
上面其实还会有一种情况，如果那个CDN节点的缓存服务器里面并没有用户要访问的内容的缓存，那么缓存服务器就需要向源服务器请求数据，然后再把数据返回给用户，并缓存数据。
