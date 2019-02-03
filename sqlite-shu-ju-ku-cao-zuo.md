Go 支持 sqlite 的驱动也比较多，但是好多都是不支持 database/sql 接口的

https://github.com/mattn/go-sqlite3 支持 database/sql 接口，基于 cgo \(关于 cgo 的知识请参看官方文档或者本书后面的章节\) 写的

https://github.com/feyeleanor/gosqlite3 不支持 database/sql 接口，基于 cgo 写的

https://github.com/phf/go-sqlite3 不支持 database/sql 接口，基于 cgo 写的

增删改成基本和mysql操作数据一样，唯一改变的是驱动的导入，然后调用 sql.Open 是采用了 SQLite 的方式打开。

`sql.Open("sqlite3", "./foo.db")`

