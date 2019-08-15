---
layout:         page
title:          【mongoDB】shell
subtitle:       
date:           2019-08-15
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

#### 定制shell提示
由于mongo shell支持JavaScript语法，我们就可以在shell中执行一些JavaScript语句。通常，每次启动都会加载的js代码，可以放在用户主目录下的 **.mongorc.js** 文件中。

默认情况下，mongo shell 是没有提示符的，这种体验不好。mongo shell 的提示符可以通过prompt变量来定制，那么我们正好可以在.mongorc.js文件中设置prompt，这样每次启动mongo shell都会自动设置好提示符。
```javascript
cmdCount = 1;
host = db.serverStatus().host;
prompt = function() {
            return db+"@"+host+":"+(cmdCount++)+"$ ";
        }
```
