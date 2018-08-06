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
C:\Users\santiago>nslookup  
默认服务器:  UnKnown  
Address:  ***  
\> set type=a  
\> www.iqiyi.com  
服务器:  UnKnown  
Address:  ***  
非权威应答:  
名称:    static.dns.iqiyi.com  
Addresses:  ***  
Aliases:  www.iqiyi.com  
\> vodguide.pps.iqiyi.com  
服务器:  UnKnown  
Address:  ***  
非权威应答:  
名称:    static.dns.iqiyi.com  
Addresses:  ***  
Aliases:  vodguide.pps.iqiyi.com  

上面的例子可以看出，www.iqiyi.com和vodguide.pps.iqiyi.com都是static.dns.iqiyi.com的一个别名。
就是说，用户访问www.iqiyi.com或者vodguide.pps.iqiyi.com，实际上最后都访问的是static.dns.iqiyi.com。
而访问static.dns.iqiyi.com，就会走爱奇艺的CDN网络。

## CDN网络
CDN网络对域名解析过程是进行调整了的。CDN提供服务的原理可以概括为以下几步：  
1. 域名重定向  
就如上面说到的，www.iqiyi.com和vodguide.pps.iqiyi.com都被重定向到static.dns.iqiyi.com。  
2. CDN要根据原始域名返回用户一个CDN缓存服务器IP地址  
原始域名就在http头里面。  
3. 返回CDN缓存服务器IP地址这里，就是CDN调整域名解析过程的关键一步了  
CDN根据全局负载均衡，以及用户地理位置等信息，返回一个合适该用户的CDN缓存服务器IP地址。让用户访问速度更快。  
4. CDN缓存服务器请求源服务器  
上面其实还会有一种情况，如果那个CDN缓存服务器里面并没有用户要访问的内容的缓存，那么那个CDN缓存服务器就需要向源服务器请求数据，然后再把数据返回给用户，并缓存数据。
