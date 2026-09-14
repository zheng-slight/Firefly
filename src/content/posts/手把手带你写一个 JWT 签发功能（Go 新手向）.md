---
title: [Go 新手向] 手把手带你写一个 JWT 签发功能
published: 2026-09-14
description: 从零到跑通，10 分钟搞懂 JWT 的签发与验证
tags: [Go, Golang, JWT, 身份认证, 新手教程]
category: 后端开发
---

# 手把手带你写一个 JWT 签发功能（Go 新手向）

刚接触 JWT 的时候，我盯着代码看了半天，每个字母都认识，连起来就不知道在干嘛。后来写多了才发现，这东西其实就那么几步，用熟了根本不用想原理。

这篇文章就一个目标：让你看完能自己写出一个能跑的 JWT 签发功能。原理点到为止，重点在"怎么写、怎么用"。

---

## 先搞清楚 JWT 长什么样

一个 JWT 就是下面这种字符串：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEyMywiZXhwIjoxNzM1Njg5NjAwLCJpc3MiOiJteS1hcHAifQ.xxxxxxxx
```

中间用两个点分成三段：

- 第一段：**头**，说明用什么算法签名
- 第二段：**载荷**，你真正想存的数据（比如用户 ID）
- 第三段：**签名**，防止别人篡改

你不需要记住里面是什么，只要知道：**前两段只是 base64url 编码，不是加密，任何人拿到都能解码查看内容；第三段才是用来验证真假的。**

所以千万别往 JWT 里塞密码、手机号这种敏感信息。

---

## 准备工作

新建一个 Go 项目，装依赖：

```bash
go mod init myjwt
go get github.com/golang-jwt/jwt/v5
```

> 别用 `dgrijalva/jwt-go`，那个库已经归档不维护了，现在用 `github.com/golang-jwt/jwt/v5`。

---

## 第一步：定义要存的数据

JWT 里的数据叫 Claims（声明）。我们要存用户 ID，就自己定义一个结构体：

```go
type MyClaims struct {
	Uid int `json:"uid"`
	jwt.RegisteredClaims
}
```

这里有两个东西：

**`Uid int`**：你自己要存的数据，想存啥就加啥字段。加 `json:"uid"` 是为了让它在 token 里显示成小写的 `uid`，不加的话就是 `Uid`，这只是格式问题，不影响使用。

**`jwt.RegisteredClaims`**：这是库自带的，里面装着过期时间、签发者这些标准字段。**必须嵌入它**，否则 token 的过期时间没法设，库也不认。

---

## 第二步：写签发函数

```go
// 仅用于本地测试，生产环境务必换成随机生成的密钥
var jwtKey = []byte("your-secret-key-please-change-this")

func SetToken(uid int) (string, error) {
	// 1. 设置过期时间
	expireTime := time.Now().Add(24 * time.Hour)

	// 2. 把数据装进 Claims
	claims := MyClaims{
		Uid: uid,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(expireTime),
			Issuer:    "my-app",
		},
	}

	// 3. 创建 token 对象，指定签名算法
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)

	// 4. 用密钥签名，拿到最终字符串
	return token.SignedString(jwtKey)
}
```

就这四步，挨个说。

**第 1 步**：算一下 24 小时后的时间，作为过期时间。

**第 2 步**：把 uid 和过期时间、签发者一起塞进 `MyClaims`。`jwt.NewNumericDate` 是把 `time.Time` 转成 JWT 需要的数字时间戳，照用就行。

**第 3 步**：`jwt.NewWithClaims` 的作用是——"我要用 HS256 算法，签这份数据"。它返回一个 token 对象，但这时候还没签名，只是把头部和载荷组装好了。

**第 4 步**：`SignedString` 用你的密钥 `jwtKey` 对前面的内容做签名，拼成最终那个三段式字符串。

**`jwtKey` 是啥？**

它就是一把钥匙，用来签名和验证。**必须保密**，上面那串只是教学用的示例，实际项目里要用一长串随机字符。

如果密钥泄露了，别人就能伪造任意 token 冒充任何用户。实际项目里建议用环境变量存：

```go
var jwtKey = []byte(os.Getenv("JWT_SECRET"))
```

记得 `import "os"`。本文下面的完整代码为了简洁仍用固定值，实际项目替换成这行即可。

---

## 第三步：验证 token

签发了还得能验，不然没用。写一个解析函数：

```go
func ParseToken(tokenStr string) (*MyClaims, error) {
	claims := &MyClaims{}

	token, err := jwt.ParseWithClaims(tokenStr, claims, func(t *jwt.Token) (interface{}, error) {
		return jwtKey, nil
	})

	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, errors.New("token 无效")
	}

	return claims, nil
}
```

`jwt.ParseWithClaims` 干三件事：

1. 把 token 字符串拆开
2. 用 `jwtKey` 验证签名对不对
3. 检查有没有过期

验证通过后，`claims.Uid` 就是你当初存进去的用户 ID。

那个回调函数 `func(t *jwt.Token) (interface{}, error)` 看起来有点怪，它其实就是告诉库"用哪个密钥验证"。现在你只要照抄就行。

**关于 `Issuer`**：签发时我们塞了 `Issuer: "my-app"`，但 `ParseWithClaims` **不会自动校验它**。它只是被存进 token 里，要不要检查由你自己决定。需要的话，在解析成功后加一句：

```go
if claims.Issuer != "my-app" {
    return nil, errors.New("签发者不匹配")
}
```

不校验也能跑，只是少了一层安全保证。

---

## 完整代码

```go
package main

import (
	"errors"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// 仅用于本地测试，生产环境务必换成随机生成的密钥
var jwtKey = []byte("your-secret-key-please-change-this")

type MyClaims struct {
	Uid int `json:"uid"`
	jwt.RegisteredClaims
}

// 签发 token
func SetToken(uid int) (string, error) {
	expireTime := time.Now().Add(24 * time.Hour)

	claims := MyClaims{
		Uid: uid,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(expireTime),
			Issuer:    "my-app",
		},
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtKey)
}

// 验证 token
func ParseToken(tokenStr string) (*MyClaims, error) {
	claims := &MyClaims{}

	token, err := jwt.ParseWithClaims(tokenStr, claims, func(t *jwt.Token) (interface{}, error) {
		return jwtKey, nil
	})

	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, errors.New("token 无效")
	}

	return claims, nil
}

func main() {
	// 签发
	tokenStr, err := SetToken(123)
	if err != nil {
		fmt.Println("签发失败:", err)
		return
	}
	fmt.Println("token:", tokenStr)

	// 验证
	claims, err := ParseToken(tokenStr)
	if err != nil {
		fmt.Println("验证失败:", err)
		return
	}
	fmt.Println("用户 ID:", claims.Uid)
}
```

跑一下，输出类似：

```
token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEyMywiZXhwIjoxNzM1Njg5NjAwLCJpc3MiOiJteS1hcHAifQ.xxxxx
用户 ID: 123
```

---

## 实际项目里怎么用

签发一般放在**登录接口**里（下面是伪代码，`user` 表示你从数据库查出来的用户）：

```go
func Login(c *gin.Context) {
	// ... 验证用户名密码，查出 user ...

	token, _ := SetToken(user.ID)
	c.JSON(200, gin.H{"token": token})
}
```

验证一般放在**中间件**里，每次请求先检查（下面用到 `strings.SplitN`，需要 `import "strings"`）：

```go
func AuthMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		authHeader := c.GetHeader("Authorization") // "Bearer xxx.yyy.zzz"

		// 取 "Bearer " 后面的部分
		parts := strings.SplitN(authHeader, " ", 2)
		if len(parts) != 2 {
			c.AbortWithStatus(401)
			return
		}

		claims, err := ParseToken(parts[1])
		if err != nil {
			c.AbortWithStatus(401)
			return
		}

		c.Set("uid", claims.Uid)
		c.Next()
	}
}
```

前端拿到 token 后，每次请求把它放在请求头里：

```
Authorization: Bearer eyJhbGci...xxxxx
```

服务端从请求头里取出来，去掉 `Bearer ` 前缀，剩下的就是 token 本身。

---

## 几个新手常踩的坑

**1. 密钥写太简单**

别用 `"secret"`、`"123456"` 这种。密钥越随机越安全，建议用 32 位以上的随机字符串。

**2. 往 token 里塞敏感信息**

JWT 前两段是 base64url 编码，不是加密。随便找个网站就能解开看。密码、身份证号这些绝对不要放。

**3. 忘了设过期时间**

不设过期时间的 token 一旦泄露就是永久有效，风险很大。一般设 1~24 小时。

**4. 用已归档的库**

`dgrijalva/jwt-go` 已停止维护，有安全问题。直接上 `github.com/golang-jwt/jwt/v5`。

**5. 解析时直接取 `strings.Split(data, " ")[1]`**

如果请求头格式不对，这里会数组越界 panic。用 `SplitN` 并判断长度更安全。

**6. 以为设了 `Issuer` 就自动校验**

`Issuer` 只是存进去，验证时不会自动检查。需要手动比对 `claims.Issuer`。

---

## 总结

JWT 签发就四步：

1. 定义 Claims 结构体（嵌入 `jwt.RegisteredClaims`）
2. 填数据
3. `NewWithClaims` 指定算法
4. `SignedString` 签名

验证就三步：

1. `ParseWithClaims` 解析并验签
2. 检查 `token.Valid`
3. 从 claims 里取你要的数据

把上面完整代码复制过去改改就能用。剩下的就是记得密钥要保密、别塞敏感信息、一定要设过期时间。

用起来其实比想象中简单。