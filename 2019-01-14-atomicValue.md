---
title: 无锁编程--atomic.Value
author: dafsic
date: 2019-01-14 06:53:16
categories: golang
tags: atomic
---

# 无锁编程--atomic.Value

## 背景
多个goroutine都会读写同一个对象时，必然会有竞争。读写锁或者互斥锁就是解决这种问题的终极方法，但同时性能也是最差的，go在atomic库提供了Value这个类型，实现无锁操作。

## 实现

{% highlight go %}
type HomeCache map[string]int

var homePageInfo atomic.Value

func updateHomeCache() error {
	data := HomeCache{}
	queryStr := "SELECT source,type,value,wihome FROM wihome_index"

	rows, _ := adb.DB.Query(queryStr)
	var source, typ, wihome int
	var value, key string
	for rows.Next() {
		rows.Scan(&source, &typ, &value, &wihome)
		key = fmt.Sprintf("%d__%d__%s", source, typ, value)
		data[key] = wihome
	}
	if rows.Err() != nil {
		return rows.Err()
	}

	homePageInfo.Store(data)
	return nil
}

func GetHomeIdx(source int, did string, mid int, node string) (idx int) {
	data := homePageInfo.Load().(HomeCache)

	var ok bool

	key := fmt.Sprintf("%d__%d__%s", source, 3, did)
	if idx, ok = data[key]; ok {
		return
	}

	return
}
{% endhighlight %}

## 注意事项
一旦向Value存储对象后，类型就固定了，这个Value之后只能存储这个类型的对象，如果存储其他对象，会发生恐慌。