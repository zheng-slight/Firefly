---
title: Go DES/3DES 加解密工具包
published: 2026-09-10
description: 一个简洁、安全、可直接落地的 DES / 3DES 工具包，含使用文档
tags: [Go, DES, 3DES, 加解密, 工具包]
category: 工具包
comment: true
---

# Go DES/3DES 加解密工具包

一个简洁、安全、可直接落地的 DES / 3DES 工具包，含使用文档。

---

## 一、代码实现

文件：`desutil/des.go`

```go
// Package desutil 提供 DES / 3DES 的 CBC 模式加解密工具。
//
// 安全提示：DES 密钥有效强度仅 56 位，已不具备现代安全性，
// 仅用于兼容旧系统。新项目请使用 AES（见 crypto/aes）。
// 若必须沿用 DES 家族，请使用 3DES（24 字节密钥）。
package desutil

import (
	"bytes"
	"crypto/cipher"
	"crypto/des"
	"crypto/rand"
	"errors"
	"io"
)

const (
	// DesKeySize 是 DES 的密钥长度（8 字节）。
	DesKeySize = 8
	// TripleDesKeySize 是 3DES 的密钥长度（24 字节）。
	TripleDesKeySize = 24
)

// ErrInvalidCipherText 表示密文长度非法或填充校验失败。
var ErrInvalidCipherText = errors.New("desutil: invalid cipher text")

// ---------- DES ----------

// DesEncrypt 使用 DES-CBC 加密明文。
// key 必须为 8 字节。返回值为 iv(8字节) + 密文。
func DesEncrypt(plain, key []byte) ([]byte, error) {
	block, err := des.NewCipher(key)
	if err != nil {
		return nil, err
	}
	return cbcEncrypt(block, plain)
}

// DesDecrypt 解密由 DesEncrypt 产生的数据（iv + 密文）。
// key 必须为 8 字节。
func DesDecrypt(data, key []byte) ([]byte, error) {
	block, err := des.NewCipher(key)
	if err != nil {
		return nil, err
	}
	return cbcDecrypt(block, data)
}

// ---------- 3DES ----------

// TripleDesEncrypt 使用 3DES-CBC 加密明文，key 必须为 24 字节。
func TripleDesEncrypt(plain, key []byte) ([]byte, error) {
	block, err := des.NewTripleDESCipher(key)
	if err != nil {
		return nil, err
	}
	return cbcEncrypt(block, plain)
}

// TripleDesDecrypt 使用 3DES-CBC 解密密文，key 必须为 24 字节。
func TripleDesDecrypt(data, key []byte) ([]byte, error) {
	block, err := des.NewTripleDESCipher(key)
	if err != nil {
		return nil, err
	}
	return cbcDecrypt(block, data)
}

// ---------- 内部通用实现 ----------

// cbcEncrypt 对明文做 PKCS5 填充后用 CBC 加密，
// 并把随机 IV 前置拼到密文头部。
func cbcEncrypt(block cipher.Block, plain []byte) ([]byte, error) {
	plain = pkcs5Padding(plain, block.BlockSize())

	iv := make([]byte, block.BlockSize())
	if _, err := io.ReadFull(rand.Reader, iv); err != nil {
		return nil, err
	}

	out := make([]byte, len(iv)+len(plain))
	copy(out, iv)
	cipher.NewCBCEncrypter(block, iv).CryptBlocks(out[len(iv):], plain)
	return out, nil
}

// cbcDecrypt 从 data 头部取出 IV，解密后做 PKCS5 去填充。
func cbcDecrypt(block cipher.Block, data []byte) ([]byte, error) {
	bs := block.BlockSize()
	if len(data) < bs || (len(data)-bs)%bs != 0 {
		return nil, ErrInvalidCipherText
	}

	iv, cipherText := data[:bs], data[bs:]
	plain := make([]byte, len(cipherText))
	cipher.NewCBCDecrypter(block, iv).CryptBlocks(plain, cipherText)

	return pkcs5UnPadding(plain)
}

// ---------- 填充 ----------

// pkcs5Padding 在尾部补齐到 blockSize 的整数倍。
// 即使已对齐也会额外补一整块（值 = blockSize），保证可逆。
func pkcs5Padding(src []byte, blockSize int) []byte {
	padding := blockSize - len(src)%blockSize
	return append(src, bytes.Repeat([]byte{byte(padding)}, padding)...)
}

// pkcs5UnPadding 校验并剥除 PKCS5 填充。
// 校验每个填充字节的值是否一致，防止伪造密文导致越界或信息泄露。
func pkcs5UnPadding(src []byte) ([]byte, error) {
	n := len(src)
	if n == 0 {
		return nil, ErrInvalidCipherText
	}
	pad := int(src[n-1])
	if pad == 0 || pad > n {
		return nil, ErrInvalidCipherText
	}
	for _, b := range src[n-pad:] {
		if int(b) != pad {
			return nil, ErrInvalidCipherText
		}
	}
	return src[:n-pad], nil
}
```

---

## 二、使用文档

### 安装

把 `desutil` 目录放进项目，`import "your/module/desutil"` 即可，无第三方依赖。

### 快速上手

```go
package main

import (
	"encoding/hex"
	"fmt"

	"your/module/desutil"
)

func main() {
	key := []byte("12345678") // DES: 8 字节
	plain := []byte("hello des")

	// 加密：返回 iv + 密文
	cipherText, err := desutil.DesEncrypt(plain, key)
	if err != nil {
		panic(err)
	}
	fmt.Println("密文(hex):", hex.EncodeToString(cipherText))

	// 解密：直接传入加密结果
	got, err := desutil.DesDecrypt(cipherText, key)
	if err != nil {
		panic(err)
	}
	fmt.Println("明文:", string(got))
}
```

### 3DES 用法

```go
key := []byte("123456789012345678901234") // 24 字节
cipherText, _ := desutil.TripleDesEncrypt([]byte("hello 3des"), key)
plain, _ := desutil.TripleDesDecrypt(cipherText, key)
```

### API 一览

| 函数                           | 入参    | 返回      | 说明          |
| ------------------------------ | ------- | --------- | ------------- |
| `DesEncrypt(plain, key)`       | key 8B  | `iv+密文` | DES-CBC 加密  |
| `DesDecrypt(data, key)`        | key 8B  | 明文      | DES-CBC 解密  |
| `TripleDesEncrypt(plain, key)` | key 24B | `iv+密文` | 3DES-CBC 加密 |
| `TripleDesDecrypt(data, key)`  | key 24B | 明文      | 3DES-CBC 解密 |

### 约定与注意事项

1. **返回格式统一为 `IV || 密文`**，解密端无需额外传 IV。
2. **IV 每次随机生成**，同明文同密钥多次加密结果不同，属正常。
3. **密文长度** = 8（IV） + 向上取整到 8 的明文长度。
4. **解码失败统一返回 `ErrInvalidCipherText`**，调用方用 `errors.Is` 判断即可。
5. 密钥长度错误会由 `des.NewCipher` / `NewTripleDESCipher` 返回错误，无需自行校验。

```go
if _, err := desutil.DesDecrypt(data, key); errors.Is(err, desutil.ErrInvalidCipherText) {
	// 密文损坏或被篡改
}
```

---

## 三、关键点回顾

- **IV 必须随机且随密文传输**，不能复用密钥当 IV。
- **PKCS5 填充即使对齐也补一整块**，否则解密时无法判断是否要去填充。
- **去填充必须逐字节校验**，否则既是 panic 隐患也是 Padding Oracle 的入口。
- **DES 已不安全**，能选 3DES 就别用 DES，能用 AES 就别用 DES 家族。

---

## 四、结语

工具包只有一个文件、四个导出函数、无外部依赖，接口对调用方友好（IV 内置、错误统一）。直接替换掉原参考代码即可，行为兼容（加密结果格式由 `密文` 变为 `IV+密文`，解密需同步更新为配套版本）。