---
layout: post
categories: 
title: Bark服务端部署文档
date: 2018-06-26 16:59:59 +0800
description: 
keywords: Bark
catalog: true
multilingual: false
tags: Bark
---

<a href="https://github.com/Finb/bark-server/blob/master/README.md">For English</a>
<br/>


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

- Serverless 
  

  默认提供 Heroku ~~免费~~ 一键部署 (2022-11-28日后收费)<br>
  [![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/finb/bark-server)<br>

  其他支持WEB路由的 serverless 服务器可以使用 `bark-server -serverless true` 开启。

  开启后， bark-server 会读取系统环境变量 BARK_KEY 和 BARK_DEVICE_TOKEN, 需提前设置好。


  | 变量名 | 填写要求 |
  | ---- | ---- |
  | BARK_KEY | 除了不能填 "push" 外，可以随便填写你喜欢的。|
  | BARK_DEVICE_TOKEN | Bark App 设置中显示的 DeviceToken，此 Token 是 APNS 真实设备 Token ,请不要泄露 |

  请注意 Serverless 模式只允许一台设备使用

- 命令行发推送

```sh
# 设置环境变量
# 下载key https://raw.githubusercontent.com/Finb/bark-server/master/deploy/AuthKey_LH4T9V5U4R_5U8LBRXG3A.p8 
# 将key文件路径填到下面
TOKEN_KEY_FILE_NAME= 
# 从 app 设置中复制 DeviceToken 到这
DEVICE_TOKEN=

#下面的不要修改
TEAM_ID=5U8LBRXG3A
AUTH_KEY_ID=LH4T9V5U4R
TOPIC=me.fin.bark
APNS_HOST_NAME=api.push.apple.com
JWT_ISSUE_TIME=$(date +%s)
JWT_HEADER=$(printf '{ "alg": "ES256", "kid": "%s" }' "${AUTH_KEY_ID}" | openssl base64 -e -A | tr -- '+/' '-_' | tr -d =)
JWT_CLAIMS=$(printf '{ "iss": "%s", "iat": %d }' "${TEAM_ID}" "${JWT_ISSUE_TIME}" | openssl base64 -e -A | tr -- '+/' '-_' | tr -d =)
JWT_HEADER_CLAIMS="${JWT_HEADER}.${JWT_CLAIMS}"
JWT_SIGNED_HEADER_CLAIMS=$(printf "${JWT_HEADER_CLAIMS}" | openssl dgst -binary -sha256 -sign "${TOKEN_KEY_FILE_NAME}" | openssl base64 -e -A | tr -- '+/' '-_' | tr -d =)
AUTHENTICATION_TOKEN="${JWT_HEADER}.${JWT_CLAIMS}.${JWT_SIGNED_HEADER_CLAIMS}"

#发送推送
curl -v --header "apns-topic: $TOPIC" --header "apns-push-type: alert" --header "authorization: bearer $AUTHENTICATION_TOKEN" --data '{"aps":{"alert":"test"}}' --http2 https://${APNS_HOST_NAME}/3/device/${DEVICE_TOKEN}

# 推送参数格式可以参考
# https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/generating_a_remote_notification
# 一定要带上 "mutable-content" : 1 ，否则推送扩展不执行，不会保存推送。
# 示例：
{
   "aps" : {
      "alert" : {
         "title" : "title",
         "subtitle" : "subtitle",
         "body" : "body"
      },
      "mutable-content" : 1
   }
}

```

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

