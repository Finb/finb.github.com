---
layout: post
title: "Kingfisher 3 源码全解读笔记"
date: 2016-10-11 16:00:41 +0800
comments: true
categories: 
---

个人笔记，不适合阅读

## Kingfisher.swift

泛型Base接受任何类型，实例化Kingfisher后将Base的实例保存在 base字段里，
{% highlight Swift %}
public final class Kingfisher<Base> {
    public let base: Base
    public init(_ base: Base) {
        self.base = base
    }
}
{% endhighlight %}

kf字段用于获取对象的 Kingfisher 实例对象，用来调用一些Kingfisher提供的功能。

如果一个类需要使用 kf 字段，则只需要实现KingfisherCompatible协议即可

Kingfisher扩展了所有遵守KingfisherCompatible协议的类，用于实例化一个Kingfisher对象。

{% highlight Swift %}
public protocol KingfisherCompatible {
    associatedtype CompatibleType
    var kf: CompatibleType { get }
}

public extension KingfisherCompatible {
    public var kf: Kingfisher<Self> {
        get { return Kingfisher(self) }
    }
}
{% endhighlight %}

例如
{% highlight Swift %}
extension Image: KingfisherCompatible { }
{% endhighlight %}

如果需要其他的类获得上面的 kf 字段功能，则直接实现 KingfisherCompatible 协议即可。
例如
{% highlight Swift %}
extension ImageView: KingfisherCompatible { }
extension Button: KingfisherCompatible { }
{% endhighlight %}

这是`装饰者模式`在Swift里的一个优雅实现，简单实用
<!--more-->