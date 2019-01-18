先获取beego库下来:

```go
go get "github.com/astaxie/beego/orm"
```

导入数据库驱动：

```go
"github.com/astaxie/beego/orm"
_"github.com/go-sql-driver/mysql"
```

```go
package main
import (
"fmt"
"github.com/astaxie/beego/orm"
_"github.com/go-sql-driver/mysql"
)
type User struct {
Id int
Name string `orm:"size(100)"`
}
func init() {
//注册驱动
//orm.RegisterDriver("mysql",orm.DR_MySQL)
//设置默认数据库
err := orm.RegisterDataBase("default","mysql","root:root@tcp(127.0.0.1:3306)/test?charset=utf8",30)
if err != nil {
panic(err)
}
//注册自定义的model
orm.RegisterModel(new(User))
//创建表
//err =orm.RunSyncdb("default",false,true)
//if err != nil {
// panic(err)
//}
}
func main(){
o := orm.NewOrm()
user := &User{Name:"zhx"}
//插入表
id,err := o.Insert(user)
if err != nil {
fmt.Println(err)
}
fmt.Println("insert id:",id)
//更新表
user.Name="金玲"
num,err := o.Update(user)
if err != nil {
fmt.Println(err)
}
fmt.Println("update num:",num)
//读取signle
u := User{Id:user.Id}
err = o.Read(&u)
fmt.Println(err)
fmt.Println(u.Name)
//删除表
//num,err = o.Delete(&u)
//fmt.Printf("NUM: %d, ERR: %v\n", num, err)
}
```

设置数据库的最大空闲连接  
orm.SetMaxIdleConns\("default",数量\)  
设置数据库的最大数据库连接（go&gt;=1.2）  
orm.SetMaxOpenConns\("default",数量\)  
开始调试  
orm.Debug = true  
插入数据  
o.insert\(\)//返回id，err  
批量插入  
o.insertMulti\(并列插入数量,slice\)//返回成功数量，err  


更新，多个指定字段（,）分开，空默认更新所有  
o.update\(接口,更新指定字段\)//更新行数，err  


查询

```
var user User
user := user{Id:1}
err := o.Read(&user)
if err == orm.ErrNoRows {
fmt.Println("查询不到")
} else if err == orm.ErrMissPK {
fmt.Println("找不到主键")
} else {
fmt.Println(user.Id, user.Name)
}
示例二
var user User
qs := o.QueryTable(user)
qs.Filter("id",1)//where id=1
qs.Filter("profile__age", 18) // WHERE profile.age = 18
示例三 where in 条件查询
qs.Filter("porofile__age__in",18)//// WHERE profile.age IN (18, 20)
示例四 更加复杂查询
qs.Filter("profile__age__in",20,18).Exclude("profile__lt",1000)// WHERE profile.age IN (18, 20) AND NOT profile_id <1000
qs.Filter("profile__age__gt",12)// WHERE profile.age >17

示例limit
qs.Limit(10,20)
```

删除

```
o.delete(&user{Id:1});//返回行，err

使用原生sql

o := orm.NewOrm()
var r orm.RawSeter
r = o.Raw("UPDATE user SET name = ? WHERE name = ?", "testing", "slene")
r.QueryRows(&user)//返回num,err
```



