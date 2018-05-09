---
layout: post
title: "iOS RunLoop 学习 &amp; 用RunLoop实现 当程序空闲时，执行某些代码"
date: 2016-01-06 15:33:25 +0800
comments: true
categories: iOS RunLoop
---

学习是看<a target='_blank' href='http://blog.ibireme.com/2015/05/18/runloop/ '>这篇博客</a>

讲的还算清楚易懂

和这篇博客类似的还有 <a target='_blank' href='http://v.youku.com/v_show/id_XODgxODkzODI0.html'>孙源的线下iOS视频 RunLoop 篇</a>。其实大致是一样的

看博文 看视频两个选一个看看。。我是两个都看了

这里我主要是分享一点用例，
<!--more-->
### 教程中提到的 AFNetworking 使用RunLoop

代码主要功能是让`线程常驻`，当有任务时，丢给这个线程就行。

主要是让 runLoop addPort .
然后runLoop 就会一直 Waiting 。
直到有任务丢给这个线程，
执行完后任务后  ，又回到Waiting 状态
<br/>

### 这里我自己写了一个RunLoop用法例子，

具体是 当程序空闲时，我们去执行一些代码。 

可能场景是某个tableView停止滑动后 ，APP是空闲状态。此时我们可以去执行一些计算还没有呈现出来的cell的高度，当tableView再次滑动时，我们就不需要额外计算Cell的高度了，让滑动更流畅。

#### 实现之前，首先有一个知识点要清楚，
就是RunLoop的`状态变化` ，我们要知道RunLoop何时变成空闲，才能在空闲时执行代码。

教程里有非常详细的描述，但我还是想用一段代码来测试一下

{% highlight objc %}
NSDictionary *keyDict = @{
    @1 : @"kCFRunLoopEntry",
    @2 : @"kCFRunLoopBeforeTimers",
    @4 : @"kCFRunLoopBeforeSources",
    @32 : @"kCFRunLoopBeforeWaiting",
    @64 : @"kCFRunLoopAfterWaiting",
    @128 : @"kCFRunLoopExit",
};

CFRunLoopActivity flags =
    kCFRunLoopEntry |
    kCFRunLoopBeforeTimers |
    kCFRunLoopBeforeSources |
    kCFRunLoopBeforeWaiting |
    kCFRunLoopAfterWaiting |
    kCFRunLoopExit |
    kCFRunLoopAllActivities;
    
CFRunLoopObserverRef runloopObserver = CFRunLoopObserverCreateWithHandler(
    kCFAllocatorDefault, flags, YES, 0,
    ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        NSLog(@"%@", keyDict[@(activity)]);
    });

CFRunLoopAddObserver(CFRunLoopGetCurrent(), runloopObserver, kCFRunLoopDefaultMode);
{% endhighlight %}

在某个UITableViewController 中的 viewDidLoad 加入这段代码
然后 滑动列表 观察打印的值

可以看到值的变化大概是

```
kCFRunLoopAfterWaiting
kCFRunLoopBeforeTimers
kCFRunLoopBeforeSources
kCFRunLoopBeforeWaiting

```

之间循环,当任务全部完成，将要waiting时，会发送一个kCFRunLoopBeforeWaiting通知

所以，我们知道RunLoop将空闲时，会发送 kCFRunLoopBeforeWaiting 通知，
告诉我们 他将要进入 waiting 状态。

所以将代码修改下 
{% highlight objc %}
//一个普通打印方法
- (void)print {
    NSLog(@"test");
}

- (void)viewDidLoad {
    CFRunLoopActivity flags = kCFRunLoopBeforeWaiting;
    CFRunLoopObserverRef runloopObserver = CFRunLoopObserverCreateWithHandler(
        kCFAllocatorDefault, flags, YES, 0,
        ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
            [self performSelector:@selector(print)
                         onThread:[NSThread mainThread]
                       withObject:nil
                    waitUntilDone:NO
                            modes:@[ NSDefaultRunLoopMode ]];
        });
    CFRunLoopAddObserver(CFRunLoopGetCurrent(), runloopObserver, kCFRunLoopDefaultMode);	 
}
{% endhighlight %}

运行程序，滑动列表时，print 将不会执行。

当停止滑动时，print将执行。

一次执行完毕后，RunLoop 将变成waiting状态，于是发送 kCFRunLoopBeforeWaiting 通知，调用我们写的Observer方法。。
方法里会调用performSelector方法  又将唤醒RunLoop 再执行一次print方法，
形成了一个`循环`。

这里可以跳出这个循环，比如任务完成了，就将Observer 移除就行 。

注意代码最后一行的CFRunLoopAddObserver方法的最后一个参数 `kCFRunLoopDefaultMode`，这个参数告诉RunLoop 这是只有`RunLoopDefaultMode` 才有的通知，

也就是说 当再次滑动列表时，Mode将切换成 `UITrackingRunLoopMode` 这个Mode不会发送 Observer 通知，所以滑动时，是不会执行print代码的

----

# 总结 如何在RunLoop空闲时执行代码
1. 监听RunLoop 的RunLoopDefaultMode  的 kCFRunLoopBeforeWaiting状态
2. 收到通知时，去执行代码。
3. 任务完成后，取消监听。

其他的RunLoop帮你搞定~~~