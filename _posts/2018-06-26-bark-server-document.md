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

如果你的数据特别敏感，请将Bark部署到私人服务器。<br>所有的数据将只在 你的手机、你的服务器、Apple推送服务器之间传输。

#### 部署:

1. 根据平台下载可执行文件:<br> <a href='https://github.com/Finb/Bark/releases'>https://github.com/Finb/Bark/releases</a><br>
或自己编译<br>
<a href="https://github.com/Finb/go-tools/blob/master/Bark.go">https://github.com/Finb/go-tools/blob/master/Bark.go</a>

2. 运行
```
./Bark_linux_amd64 -ip=0.0.0.0 -port=80 
```
你可能需要
```
chmod +x Bark_linux_amd64
```
3. 测试
```
curl http://0.0.0.0/ping
```
4. 在APP端填入你的服务器IP或域名

#### 推送证书:

* 当你需要集成Bark到自己的系统或重新实现后端代码时可能需要推送证书<br>
证书密码: bp<br>
有效期到: 2019-03-08<br>
 <a href="https://github.com/Finb/Bark/releases/download/1.0.0/cert-20190308.p12">https://github.com/Finb/Bark/releases/download/1.0.0/cert-20190308.p12</a>
 

#### 其他:

1. APP端负责将<a href="https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1622958-application">DeviceToken</a>发送到服务端。 <br>服务端收到一个推送请求后，将发送推送给Apple服务器。然后手机收到推送

2. 服务端代码: <a href='https://github.com/Finb/go-tools/blob/master/Bark.go'>https://github.com/Finb/go-tools/blob/master/Bark.go</a><br>

3. App代码: <a href="https://github.com/Finb/Bark">https://github.com/Finb/Bark</a>

