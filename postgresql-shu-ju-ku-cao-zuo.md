Go 实现的支持 PostgreSQL 的驱动也很多，因为国外很多人在开发中使用了这个数据库。

https://github.com/lib/pq 支持 database/sql 驱动，纯 Go 写的

https://github.com/jbarham/gopgsqldriver 支持 database/sql 驱动，纯 Go 写的

https://github.com/lxn/go-pgsql 支持 database/sql 驱动，纯 Go 写的

go实现postgreSql增删改查

```go
package main

import (
    "database/sql"
    "fmt"

    _ "github.com/lib/pq"
)

func main() {
    db, err := sql.Open("postgres", "user=astaxie password=astaxie dbname=test sslmode=disable")
    checkErr(err)

    // 插入数据
    stmt, err := db.Prepare("INSERT INTO userinfo(username,department,created) VALUES($1,$2,$3) RETURNING uid")
    checkErr(err)

    res, err := stmt.Exec("astaxie", "研发部门", "2012-12-09")
    checkErr(err)

    // pg 不支持这个函数，因为他没有类似 MySQL 的自增 ID
    // id, err := res.LastInsertId()
    // checkErr(err)
    // fmt.Println(id)

    var lastInsertId int
    err = db.QueryRow("INSERT INTO userinfo(username,departname,created) VALUES($1,$2,$3) returning uid;", "astaxie", "研发部门", "2012-12-09").Scan(&lastInsertId)
    checkErr(err)
    fmt.Println("最后插入id =", lastInsertId)

    // 更新数据
    stmt, err = db.Prepare("update userinfo set username=$1 where uid=$2")
    checkErr(err)

    res, err = stmt.Exec("astaxieupdate", 1)
    checkErr(err)

    affect, err := res.RowsAffected()
    checkErr(err)

    fmt.Println(affect)

    // 查询数据
    rows, err := db.Query("SELECT * FROM userinfo")
    checkErr(err)

    for rows.Next() {
        var uid int
        var username string
        var department string
        var created string
        err = rows.Scan(&uid, &username, &department, &created)
        checkErr(err)
        fmt.Println(uid)
        fmt.Println(username)
        fmt.Println(department)
        fmt.Println(created)
    }

    // 删除数据
    stmt, err = db.Prepare("delete from userinfo where uid=$1")
    checkErr(err)

    res, err = stmt.Exec(1)
    checkErr(err)

    affect, err = res.RowsAffected()
    checkErr(err)

    fmt.Println(affect)

    db.Close()

}

func checkErr(err error) {
    if err != nil {
        panic(err)
    }
}
```

postgreSql是通过$1,$2这种方式来指定要传递的参数，而不是mysql中的？ 另外在 sql.Open中的 dsn 信息的格式也与 MySQL 的驱动中的 dsn 格式不一样，所以在使用时请注意它们的差异。

还有pg不支持lastInserId函数，postgreSql内部没有实现类似mysql的自增id返回，其他几乎一样

