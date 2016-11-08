---
layout: post
title: Go数据库基本操作
comments: true
archive: true
tag: [GO, database]
---
最近一段时间在做go语言的数据库驱动开发，趁着刚刚鼓捣完还算熟悉，于是赶着总结一下。

## 增删改查
先来看一段简单的数据库查询操作的代码，了解一下go语言数据库操作的基本接口。本地数据库服务器为mysql，首先下载mysql驱动程序，`go get github.com\go-sql-driver\mysql`。

~~~~~go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	db, err := sql.Open("mysql", "watoud:watoud@/testdb")
	if err != nil {
		fmt.Println("open db failed, ", err.Error())
		return
	}
	defer db.Close()

	rows, err := db.Query("select id,name,age from test_table")
	if err != nil {
		fmt.Println("query failed, ", err.Error())
		return
	}

	var id, age int
	var name string
	for rows.Next() {
		rows.Scan(&id, &name, &age)
		fmt.Println("id:", id, "name:", name, "age:", age)
	}
}
~~~~~

上面的代码中首先引入mysql驱动包，`import _ "github.com/go-sql-driver/mysql"`，执行数据库驱动初始化操作，向系统中注册mysql driver。然后打开数据库连接，`db, err := sql.Open("mysql", "watoud:watoud@/testdb")`，在执行`sql.Open`时系统并不会立即去建立连接，而是在需要的时候才去真正的创建连接。db创建好之后就可以执行相关的数据库操作了，比如query、insert、delete、update等，使用`Rows.Scan`获取操作返回的结果集。下面看一个数据插入操作。

~~~~~~~go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	db, err := sql.Open("mysql", "watoud:watoud@/testdb")
	if err != nil {
		fmt.Println("open db failed, ", err.Error())
		return
	}
	defer db.Close()

	result, err := db.Exec("insert into test_table(id,name,age) values(10002,'sz',30)")
	if err != nil {
		fmt.Println("exec insert operation failed, ", err.Error())
		return
	}

	affected, err := result.RowsAffected()
	if err != nil {
		fmt.Println("failed to get affected rows,  ", err.Error())
		return
	}
	fmt.Println("RowsAffected:", affected)
}
~~~~~~~


更多示例请参看官方网站：https://golang.org/pkg/database/sql/#pkg-examples

## 预处理
在执行数据库操作的时候，很多情况下需要动态地设置一些参数，这个时候就需要用到预处理了。

~~~~~go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	db, err := sql.Open("mysql", "watoud:watoud@/testdb")
	if err != nil {
		fmt.Println("open db failed, ", err.Error())
		return
	}
	defer db.Close()

	stmt, err := db.Prepare("select id,name,age from test_table where id = ? and age < ?")
	if err != nil {
		fmt.Println("failed to prepare sql,", err.Error())
		return
	}

	rows, err := stmt.Query(10001, 40)
	if err != nil {
		fmt.Println("failed to execute query,", err.Error())
	}

	var id, age int
	var name string
	for rows.Next() {
		rows.Scan(&id, &name, &age)
		fmt.Println("id:", id, "name:", name, "age:", age)
	}
	// 此处不需要执行stmt.close操作，当Next返回为false时，系统会自动去关闭stmt以及rows
}
~~~~~

## 事务处理
当多个数据库操作需要作为一个整体进行执行的时，这个时候就需要用到事务处理接口了，具体的事务级别由驱动程序决定。

~~~~~go
package main

import (
	"database/sql"
	"fmt"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	db, err := sql.Open("mysql", "watoud:watoud@/testdb")
	if err != nil {
		fmt.Println("open db failed, ", err.Error())
		return
	}
	defer db.Close()

	insertInfo, _ := db.Prepare("insert into test_table(id,name,age) values(?,?,?)")
	tx, err := db.Begin()
	if err != nil {
		fmt.Println("failed to start a transaction,", err.Error())
		return
	}

	_, err = tx.Stmt(insertInfo).Exec(10003, "nj", 35)
	if err != nil {
		fmt.Println("failed to execute insert operation,", err.Error())
		return
	}

	err = tx.Commit()
	if err != nil {
		fmt.Println("failed to commit db operations,  ", err.Error())
		return
	}
}
~~~~~

## 参数设置
go数据库提供了几个参数设置接口，可以设置连接的最大存活时间、连接池中最多的闲置连接个数、数据库允许打开的最多连接数。 

* DB.SetConnMaxLifetime
* DB.SetMaxIdleConns
* DB.SetMaxOpenConns

当MaxOpenConns大于0且比MaxIdleConns小时，MaxIdleConns将减少为MaxOpenConns的值。









