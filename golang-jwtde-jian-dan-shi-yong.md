# golang JWT的简单使用

JWT是json web token缩写。它将用户信息加密到token里，服务器不保存任何用户信息。服务器通过使用保存的密钥验证token的正确性，只要正确即通过验证。  
JWT和session有所不同，session需要在服务器端生成，服务器保存session,只返回给客户端sessionid，客户端下次请求时带上sessionid即可。因为session是储存在服务器中，有多台服务器时会出现一些麻烦，需要同步多台主机的信息，不然会出现在请求A服务器时能获取信息，但是请求B服务器身份信息无法通过。JWT能很好的解决这个问题，服务器端不用保存jwt，只需要保存加密用的secret,在用户登录时将jwt加密生成并发送给客户端，由客户端存储，以后客户端的请求带上，由服务器解析jwt并验证。这样服务器不用浪费空间去存储登录信息，不用浪费时间去做同步。

# JWT构成

### 一、Header\(头部\)

Jwt的头部承载两部分信息：

声明类型，这里是jwt  
声明加密的算法 通常直接使用 HMAC SHA256

### 二、 Playload\(载荷又称为Claim\)

playload可以填充两种类型数据  
1.标准中注册的声明：

```
iss: 签发者
```

```

sub: 面向的用户

aud: 接收方

exp: 过期时间

nbf: 生效时间

iat: 签发时间

jti: 唯一身份标识
```

2.自定义数据

### 三、Signature（签名）

签名的算法：

```go

HMACSHA256(
    base64UrlEncode(header) + "." +
    base64UrlEncode(payload),
    secret
)


```

# 代码实现

```go
type UserInfo struct {
    ID uint64
    Username string
}


```

1.生成token

```go
func CreateToken(user *UserInfo)(tokenss string,err error){
      //自定义claim
    claim := jwt.MapClaims{
        "id":       user.ID,
        "username": user.Username,
        "nbf":      time.Now().Unix(),
        "iat":      time.Now().Unix(),
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256,claim)
    tokenss,err  = token.SignedString([]byte(viper.GetString("token.secret"))) 
    return
}


```

用户登录后获得的ID和username可以保存在自定义的mapclaim中，在mapcalim中还可以存放需要的用户信息

2.解析token

```go
func secret()jwt.Keyfunc{
    return func(token *jwt.Token) (interface{}, error) {
        return []byte(viper.GetString("token.secret")),nil
    }
}

func ParseToken(tokenss string)(user *UserInfo,err error){
    user = 
&
UserInfo{}
    token,err := jwt.Parse(tokenss,secret())
    if err != nil{
        return
    }
    claim,ok := token.Claims.(jwt.MapClaims)
    if !ok{
        err = errors.New("cannot convert claim to mapclaim")
        return
    }
    //验证token，如果token被修改过则为false
    if  !token.Valid{
        err = errors.New("token is invalid")
        return
    }

    user.ID =uint64( claim["id"].(float64))
    user.Username = claim["username"].(string)
    return
}


```

用户请求时带上token，服务器解析token后可以获得其中的用户信息，如果token有任何改动，都无法通过验证.

