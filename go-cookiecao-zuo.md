通过net/http 包中的setcookie来设置

http.SetCookie\(w ResponseWrite,cookie \*Cookie\)

例子：

```go
expiration := time.New()
expiration = expiration.Add(1,0,0)
cookie := http.Cookie{Name:"username",Value:"astaxie",Expires:expiration}
http.SetCookie(w,&cookie)
```



读取cookie

```go
cookie,_ := r.Cookie("username")
```

读取全部

```go
for _,cookie := range r.Cookies(){
        fmt.Fprint(w, cookie.Name)
}
```



