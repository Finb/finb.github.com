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

<a href="https://github.com/Finb/bark-server/blob/master/README.md">For English</a>
<br/>

### 公共服务器（api.day.app）6月6服务不可用说明
6月6日18：30分服务器受到攻击，推送服务暂时不可用。<br>
强烈建议`搭建自己的后端服务器`来提供推送服务，可以保障稳定的推送服务。<br>如遇到自建后端方面的问题，可以发邮件或此页面评论区（需翻墙）找我询问。 <br>

### iOS14.5 不能复制推送的问题

参数 automaticallyCopy 自动复制推送内容功能在 iOS14.5 失效，可暂时先下拉推送或在锁屏界面左滑推送点查看，点击复制按钮来复制推送内容。


### Bark是啥？

<a href="https://www.v2ex.com/t/467407">https://www.v2ex.com/t/467407</a><br>
[使用教程](https://github.com/Finb/Bark/blob/master/README.md)

### 隐私保护:
如果你的数据特别敏感，请将Bark部署到私人服务器。<br>所有的数据将只在 你的手机、你的服务器、Apple推送服务器之间传输。  
   
历史消息通过 NotificationServiceExtension 扩展，在收到推送时将推送信息保存在本地，不会经过其他任何设备。  
历史记录仅由个人iCloud私有库进行同步。  
可以确保你产生的任何通知，将只留在你的设备与你的iCloud中  

### 安装:

- Docker
```
docker run -dt --name bark -p 8080:8080 -v `pwd`/bark-data:/data finab/bark-server
```

- Docker-Compose 
```
mkdir bark && cd bark
curl -sL https://git.io/JvSRl > docker-compose.yaml
docker-compose up -d
```
- 手动安装

1. 根据平台下载可执行文件:<br> <a href='https://github.com/Finb/bark-server/releases'>https://github.com/Finb/bark-server/releases</a><br>
或自己编译<br>
<a href="https://github.com/Finb/bark-server">https://github.com/Finb/bark-server</a>

2. 运行
```
./bark-server_linux_amd64 -addr 0.0.0.0:8080 -data ./bark-data
```
3. 你可能需要
```
chmod +x bark-server_linux_amd64
```
请注意 bark-server 默认使用 /data 目录保存数据，请确保 bark-server 有权限读写 /data 目录，或者你可以使用 `-data` 选项指定一个目录

### 使用
```
curl http://0.0.0.0:8080/ping
```
Ping成功后，在APP端填入你的服务器IP或域名

### 推送证书:

* 当你需要集成Bark到自己的系统或重新实现后端代码时可能需要推送证书<br>
有效期到: 永久<br>
Key ID: LH4T9V5U4R <br>
TeamID: 5U8LBRXG3A <br>
<a href="https://github.com/Finb/bark-server/releases/download/v1.0.2/AuthKey_LH4T9V5U4R_5U8LBRXG3A.p8">AuthKey_LH4T9V5U4R_5U8LBRXG3A.p8</a>

### 其他:

1. APP端负责将<a href="https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1622958-application">DeviceToken</a>发送到服务端。 <br>服务端收到一个推送请求后，将发送推送给Apple服务器。然后手机收到推送

2. 服务端代码: <a href='https://github.com/Finb/bark-server'>https://github.com/Finb/bark-server</a><br>

3. App代码: <a href="https://github.com/Finb/Bark">https://github.com/Finb/Bark</a>

