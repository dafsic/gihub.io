---
title: 无锁编程--优先级channel
author: dafsic
date: 2019-01-14 06:53:16
categories: golang
tags: channel
---

# 无锁编程之优先级队列

## 背景
处理channel中的数据过程中，涉及到配置的变更，配置变更后，数据的处理方式会跟随改变。在配置可以热加载的应用中，如何更新配置结构并应用新配置，而不产生竞争。

## 实现
方法有很多，锁、原子操作、atomic.Value等，这里使用优先级队列的方式实现如下：

{% highlight go %}
func filter(srcC <-chan *T) <-chan *T {
	c := make(chan *T, 1024)

	conf := GetConfig()

	var dataP *T
	go func() {
		for {
			select {  //select + default 实现优先级channel
			case <-eventC:
				conf = GetConfig()
			default:
				dataP = <-srcC
				if _, ok := conf[dataP.Id]; ok {
					c <- dataP
				}
			}
		}
	}()
	return c
}
{% endhighlight %}

## 注意

如果default里的channel中的数据如果来的不是很频繁，会阻塞在这里，导致第一优先级的数据延迟。如果不想延迟，就要在default里再加一层select，不阻塞读取，
但这样会导致两个队列都无数据时，cpu空转，占用很大cpu资源。
