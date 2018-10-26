---
layout: post
title: "V2ex iOS客户端 BUG &amp; 建议"
date: 2016-02-01 18:54:25 +0800
comments: true
categories: 
---

你可以给我写邮件或在当前页面评论处反馈问题、提出建议。  
邮箱：<to@day.app> 。  
评论：<a href='#disqus_thread'>评论 (Disqus，需翻墙)</a>



下面列出一些常见的问题与答复

#### 0. iPhone Xs / Max 适配
请使用Testflight更新适配版本 （新版暂时没有上架App Store）<br>
TestFlight 邀请链接  <a href="https://testflight.apple.com/join/HEMrBRbv" target="_blank">https://testflight.apple.com/join/HEMrBRbv</a>

在你刷新帖子列表时会判断需不需要签到，如果需要，则会自动签到，成功后会有一个提示。

#### 1. 签到功能在哪

在你刷新帖子列表时会判断需不需要签到，如果需要，则会自动签到，成功后会有一个提示。

#### 2. 是否会偷偷上传用户隐私数据

当然是不会的，你可以用你的监控工具监控软件的网络请求。
软件只会请求 [v2ex.com](v2ex.com)
与 Twitter旗下的 [crashlytics.com](crashlytics.com)
并且，你可以[下载源代码](https://github.com/Finb/V2ex-Swift)自行编译使用

#### 3. 为什么没有发帖与搜索

这个软件最初的想法是提供一个在地铁上或空闲时`阅读`V2EX的选择，并不是要取代网站使用
当然，最主要的原因是还没写 

