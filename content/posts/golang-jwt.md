---
title: "Golang 中使用 JWT 进行认证"
date: 2021-05-10T13:19:47+08:00
archives: 
    - 2021
tags:
    - Golang
    - 后端
image: /images/golang-jwt.webp
draft: false
---

JSON Web Token (JWT) 是一种目前流行的跨域认证方案，JWT 是基于 JSON 的经过签名的 Token，可以在进行验证的同时附带身份信息，保证传输的信息没有被修改，对于前后端分离项目很有帮助。

> JSON Web Token (JWT) 是一个开放标准 (RFC 7519)，它定义了一种紧凑的、自包含的方式，用于作为 JSON 对象在各方之间安全地传输信息。该信息可以被验证和信任，因为它是数字签名的。

## 原理
JWT 的认证流程是，用户登录成功后，服务端生成一个 JWT 格式的 Token，发回给用户，这个 Token 包含了用于标记用户的信息，此后用户与服务端通信时，都携带这个 Token，服务端就能靠这个 Token 确定用户的身份。

如上面提到，这个 Token 是有签名的，所以可以保证数据不被篡改，是值得信任的。

JWT 由三部分组成，它们之间用圆点(.)相连。分别为：
- Header（头部）
- Payload（负载）
- Signature（签名）

最后生成的字符串大概就是这样子的：
```
xxx.yyy.zzz
```
下面简单说一下这三个部分。

### Header
头部主要用来描述一些元数据。
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
主要就包含这两个字段，`alg`是所使用的签名算法，服务端会使用该算法签名或验证 Token，`typ`就是类型了，固定为`JWT`。

最后使用 Base64URL 算法把这个 JSON 对象转换为字符串。

可以看到这部分是没有加密的，所以不要放置敏感信息。

### Payload
负载部分用来存放实际传递的数据，也就是关于用户的数据，JWT 官方提供了七个预定义的字段：
```json
{
  "iss": "签发人",
  "exp": "过期时间",
  "sub": "主题",
  "aud": "受众",
  "nbf": "生效时间",
  "iat": "签发时间",
  "jti": "编号"
}
```
除此之外，我们可以自己定义需要的字段：
```json
{
  "uid": 1001,
  "name": "张三",
  "iat": 1516239022
}
```
最后使用 Base64URL 算法把这个 JSON 对象转换为字符串。

可以看到这部分是没有加密的，所以不要放置敏感信息。

### Signature
最后这部分是对前两个部分的信息进行签名，以防止数据被非法篡改。

首先，自己生成一个密钥，放在服务端，不能被泄露。使用 Header 里指定的签名算法，按以下规则生成签名：
```
签名算法(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  your-secret
)
```

## 使用方式
前端获取到 JWT 后，一般储存在`localStorage`里面，当需要与服务端通信时，都带上这个 Token。

### HTTP(S)
常规的 HTTP 接口一般放在请求头的`Authorization`里面。
```
Authorization: Bearer <token>
```

### WebSocket
对于 WebSocket 协议，浏览器内置的 API 似乎无法设置请求头，这里我想到以下几种方案：
- 使用第三方 WebSocket 客户端，支持添加`Authorization`请求头
- 在 WebSocket 连接成功后发送 JWT，服务端验证不通过后关闭连接
- 使用 WebSocket 子协议放置 JWT

重点说一下最后一种方法，前端在创建 WebSocket 时可以这样传参：
```js
new WebSocket(url, ['token', 'xxx.yyy.zzz'])
```
然后浏览器的请求头就会在`Sec-WebSocket-Protocol`附带上我们的 Token 信息：
```
Sec-WebSocket-Protocol: token, xxx.yyy.zzz
```
服务端就可以取得 Token 判断是否允许建立连接。

注意：使用这种方法后端也需要开启支持子协议 token。

## 示例
我使用的是 [jwt-go](https://github.com/dgrijalva/jwt-go) 这个库。

### 创建 JWT
一般在用户登录成功后创建 JWT 返回给用户，注意以`your`开头的地方，是需要自己定义的。
```go
func Login() string {
    // 验证登录信息
    var user User
    ...
    
    // 创建 Token
    token := jwt.NewWithClaims(jwt.GetSigningMethod("HS256"), YourClaims{
		user.ID,
		user.UserName,
		jwt.StandardClaims{
			IssuedAt: jwt.Now(),
		},
	})
	// 签名，把 Token 转换为字符串
	tokenString, err := token.SignedString("yourKey")
	if err != nil {
		panic(err)
	}
	return tokenString
}
```

### 验证 JWT
客户端与服务端后续通信会附带 Token 过来，此时服务端这边需要验证这个 Token 的有效性。本文以 Gin 框架为例。
```go
func CurrentUser() gin.HandlerFunc {
	return func(c *gin.Context) {
		tokenString := c.GetHeader("Authorization")
		if tokenString == "" {
			// Websocket认证
			tokenString = c.GetHeader("Sec-WebSocket-Protocol")
		}
		if strings.HasPrefix(tokenString, "Bearer") {
			tokenString = strings.TrimPrefix(tokenString, "Bearer")
		} else if strings.HasPrefix(tokenString, "token") {
			tokenString = strings.TrimPrefix(tokenString, "token")
		}
		tokenString = strings.TrimSpace(tokenString)
		if tokenString != "" {
			if token, err := jwt.ParseWithClaims(tokenString, &YourClaims{}, 
			    jwt.KnownKeyfunc(jwt.GetSigningMethod("HS256"), "yourKey")); 
			    err == nil && token.Valid {
			    // 验证成功
				if claims, ok := token.Claims.(*YourClaims); ok {
				    // 用户 ID
				    claims.UID
				    // 用户名
				    claims.Name
				}
			}
		}
		c.Next()
	}
}
```
