---
title: golang操作mysql教程
date: 2016-09-08 19:06:20
categories: 语言
tags: [golang, mysql]
---
在go语言里访问sql数据库，一般都是用 database/sql 这个包

#### Overview

访问db使用 sql.DB，使用这个类型可以创建statements, transactions，执行查询和获取结果。但需要注意，sql.DB不是一个数据库连接。

sql.DB完成的任务：

* 依赖驱动去打开和关闭真正的数据库连接
* 按需管理连接池

#### 数据库驱动

进行任何数据库操作都需要先引入驱动包:

``` golang
import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)
```
这里匿名地引入了一个mysql驱动包，下划线符号使驱动包中的任何符号都在该文件中不可见。

#### 建立连接

先上代码：
``` golang
func main() {
	db, err := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/hello")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
}
```
以上：
* Open语句的第一个参数指定数据库类型，还可以使用 github.com/mattn/go-sqlite3  或 github.com/lib/pq
* Open语句的第二个参数指定数据库连接信息
* 记得必须要进行错误处理
* 使用defer 安全关闭连接

如果需要延迟连接数据库，可以先检查数据库能否连接成功，如下:
``` golang
err = db.Ping()
if err != nil {
	/// do something here
}
```

不要频繁的打开和关闭数据库，最好，为每个数据库只创建一个 sql.DB，如果需要使用就传过去就行了。
如果不合理使用sql.DB，就会很快耗尽网络资源，或挤压很多处于 TIME_WAIT状态的连接。

#### 从数据库获取数据

``` golang
var (
	id int
	name string
)
rows, err := db.Query("select id, name from users where id = ?", 1)
if err != nil {
	log.Fatal(err)
}
defer rows.Close()
for rows.Next() {
	err := rows.Scan(&id, &name)
	if err != nil {
		log.Fatal(err)
	}
	log.Println(id, name)
}
err = rows.Err()
if err != nil {
	log.Fatal(err)
}
```

#### prepare查询

``` golang
stmt, err := db.Prepare("select id, name from users where id = ?")
if err != nil {
		log.Fatal(err)
}
defer stmt.Close()
rows, err := stmt.Query(1)
if err != nil {
		log.Fatal(err)
}
defer rows.Close()
for rows.Next() {
		// ...
}
if err = rows.Err(); err != nil {
		log.Fatal(err)
}
```

#### 单行查询

``` golang
var name string
err = db.QueryRow("select name from users where id = ?", 1).Scan(&name)
if err != nil {
		log.Fatal(err)
}
fmt.Println(name)
```

#### prepare单行查询

``` golang
stmt, err := db.Prepare("select name from users where id = ?")
if err != nil {
		log.Fatal(err)
}
var name string
err = stmt.QueryRow(1).Scan(&name)
if err != nil {
		log.Fatal(err)
}
fmt.Println(name)
```
#### 更新数据

``` golang
stmt, err := db.Prepare("INSERT INTO users(name) VALUES(?)")
if err != nil {
		log.Fatal(err)
}
res, err := stmt.Exec("Dolly")
if err != nil {
		log.Fatal(err)
}
lastId, err := res.LastInsertId()
if err != nil {
		log.Fatal(err)
}
rowCnt, err := res.RowsAffected()
if err != nil {
		log.Fatal(err)
}
log.Printf("ID = %d, affected = %d\n", lastId, rowCnt)
```

注意不要用 query更新数据，即使你不关心返回结果，看下面两个句子，一对一错:

``` golang
_, err := db.Exec("DELETE FROM users")  // OK
_, err := db.Query("DELETE FROM users") // BAD
```

#### 事务


本文翻译自: [Go database/sql tutorial](http://go-database-sql.org/index.html)
