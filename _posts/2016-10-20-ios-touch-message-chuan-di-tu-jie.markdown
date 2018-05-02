---
layout: post
title: "iOS Touch message 传递图解"
date: 2016-10-20 15:20:39 +0800
comments: true
categories: 
---
<!--more-->
![](/pics/TouchMessage.png)



1. 系统会通过hitTest的方法寻找响应链，完成之后会形成上图模型。

2. 有了模型之后就会发生图上的三个步骤  
第一步：系统会将所有的 Touch message 优先发送给 关联在响应链上的全部手势。手势根据Touch序列消息和手势基本规则更改自己的状态（有的可能失败，有的可能识别等等）。如果没有一个手势对Touch message 进行拦截（拦截:系统不会将Touch message 发送给响应链顶部响应者)，系统会进入第二步  
第二步：系统将Touch message 发送给响应链 顶部的 视图控件，顶部视图控件这个时候就会调用Touch相关的四个方法中的某一个。之后进入自定义Touch message转发  
第三步：自定义Touch message转发可以继承UIResponser的四个Touch函数做转发。

###举例说明

aView 有一个子视图 bView ,aView 有 一个Tap手势。  
点击bView,   
bView 将收到 touchesBegan  
然后aView 手势Tap手势识别成功，执行手势   
然后 bView 收到 touchesCancelled  

经过我的测试，如果bView 是`UIButton`(或其他能响应事件的控件) 则不同，早期版本的UIButton 和UIView 一样，点击操作会失效。只会收到touchesBegan touchesCancelled  
高版本的iOS系统中，UIButton 内部估计也是使用手势，`不会被其他手势截断` (猜测)

