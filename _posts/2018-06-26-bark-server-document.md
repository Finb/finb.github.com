---
layout: post
categories: 
title: Bark服务端部署文档
date: 2018-06-26 16:59:59 +0800
description: 
keywords: 
catalog: true
multilingual: false
tags: 
---

如果你的数据特别敏感，请将Bark部署到私人服务器。<br>所有的数据将只在 你的手机 <-> 你的服务器 <-> Apple推送服务器 传输。

部署:

1. 根据平台下载可执行文件:<br> <a href='https://github.com/Finb/Bark/releases'>https://github.com/Finb/Bark/releases</a>

2. 运行
```
./Bark -ip=0.0.0.0 -port=80 
```
3. 在APP端填入你的服务器IP或域名


注意:

1. APP端负责将<a href="https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1622958-application">DeviceToken</a>发送到服务端。 <br>服务端收到一个推送请求后，将发送推送给Apple服务器。然后手机收到推送

2. 服务端代码:<a href='https://github.com/Finb/go-tools/blob/master/Bark.go'>https://github.com/Finb/go-tools/blob/master/Bark.go</a><br>
代码中的证书已失效，请替换成自己的证书
