---
title: 闭包--事务化插入数据
author: dafsic
date: 2019-01-11 02:14:14
categories: golang
tags: [closure,database]
---

# 闭包之事物化插入数据

## 背景
将队列中的数据持久化到数据库中时，不能每来一条数据就一次数据插入，肯定要批量插入。

## 实现

{% highlight go %}
import "database/sql"

func beTransaction(db *sql.DB, stmtStr string, n int) (func(...interface{}) error, func() error) {
	i := 1
	tx, _ := db.Begin()
	txstmt, _ := tx.Prepare(stmtStr)

	return func(args ...interface{}) error {
			var err error
			_, err = txstmt.Exec(args...)
			if err != nil {
				return err
			}
			i++
			if i > n {
				err = tx.Commit()
				if err != nil {
					return err
				}
				i = 1
				tx, _ = db.Begin()
				txstmt, _ = tx.Prepare(stmtStr)
			}
			return err
		},
		func() error {
			err := tx.Commit()
			if err != nil {
				logger.Println("[error]", err)
			}
			return err
		}
}
{% endhighlight %}

## 使用方法
* beTransaction（）需要三个参数，数据库句柄、预处理语句、多少条数据一起commit。
* 返回的第一个函数用来插入数据，字段个数要与预处理语句一致，当插入数到达n时，自动提交，并创建新的事务。
* 返回的第二个函数用来处理最后一次提交。

## 用例

{% highlight go %}
	stmtStr := "INSERT INTO table1(hour,mid,did,ufrom) values(?,?,?,?)"
	insertFunc, lastCommit := beTransaction(db, stmtStr, 1000)
	defer lastCommit()

	for k := range pbBuf {
		fileds := strings.Split(k, ":")
		hour, _ := strconv.ParseInt(fileds[0], 10, 8)
		mid, _ := strconv.ParseInt(fileds[1], 10, 32)
		did, _ := strconv.ParseInt(fileds[2], 10, 32)
		ufrom, _ := strconv.ParseInt(fileds[4], 10, 8)

		err = insertFunc(hour, mid, did, ufrom)
		if err != nil {
			logger.Println("[error]", err)
			os.Exit(34)
		}
	}


{% endhighlight %}