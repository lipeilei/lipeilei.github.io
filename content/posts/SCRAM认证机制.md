---
title: "SCRAM认证机制"
date: 2023-11-03T17:03:51+08:00
lastmod: 2023-11-03T17:03:51+08:00
draft: false
tags: 
- 认证机制
---

# 简介
SCRAM是一套包含服务器和客户端双向确认的用户认证体系，配合信道加密可以比较好的抵御中间人、拖库、伪造等攻击。
SCRAM是密码学中的一种认证机制，全称Salted Challenge Response Authentication Mechanism（加盐挑战响应验证机制）。它适用于使用基于“用户名：密码”这种简单认证模型的连接协议。SCRAM是一个抽象的机制，在其设计中需要用到一个哈希函数，这个哈希函数是客户端和服务端协商好的，包含在具体的机制名称中。比如SCRAM-SHA1，使用SHA1作为其哈希函数。

SCRAM是一套包含服务器和客户端双向确认的用户认证体系，配合信道加密可以比较好的抵御中间人、拖库、伪造等攻击。

# 相关字段

定义：

- `Hash(content)`指安全Hash函数，如SHA256。
- `HMAC()`指使用`Hash(content)`作为基础实现的HMAC函数，如`HMAC(SHA256,Key,Content)`。
- `UserIdentify`：仅用于登录签名的用户身份标识字符串，每次登录时可根据业务任意选用用户名、手机号、邮箱、三方ID等，下次登录可以选另一种，仅作为本次通讯的标识之一使用。
- `Password` = Hash(_明文密码_)
- `Salt`：随机生成的盐。
- `Iteration`：加盐计算时的迭代次数。
- `SaltedPassword` = pbkdf2(`Password`, `Salt`, `Iteration`)，已加盐的密码。
- `ClientNonce`：客户端在Step1时随机生成的字符串，用于对本次交互签名。
- `ServerNonce`：服务器在Step1时随机生成的字符串，用于对本次交互签名。
- `MixNonce` = `ClientNonce` | `ServerNonce`，`ClientNonce`和`ServerNonce`的_按位或_结果。
- `ServerPub`：用于服务器签名的公钥，不用保密，但也别改。
- `ClientPub`：用于客户端签名的公钥，不用保密，但也别改。
- `ServerSignedPassword` = HMAC(`ServerPub`, `SaltedPassword`)
- `ClientSignedPassword` = HMAC(`ClientPub`, `SaltedPassword`)
- `HashedClientSignedPassword` = Hash(`ClientSignedPassword`)
- `Auth` = `UserIdentify` + `ClientNonce` + `Salt` + `MixNonce` + `MixNonce`，直接连接即可，用于构造本次通讯上下文的签名。
- `SignedAuth` = HMAC(`Auth`, `HashedClientSignedPassword`)
- `ServerProof` = HMAC(`ClientSignedPassword`, `Auth`)，服务器给客户端的_身份证明_。
- `ClientProof` = `ClientSignedPassword` XOR `SignedAuth`，客户端给服务器的_身份证明_。

# 工作流程

## Client-Step1

客户端发送`UserIdentify`、`ClientNonce`给服务器端。

## Server-Step1

1. 服务器根据`UserIdentify`读取`HashPasswordClient`、`PasswordServer`、`Salt`、`Iteration`。
2. 生成`ServerNonce`、`MixNonce`。
3. 将`Salt`、`Iteration`、`MixNonce`返回给客户端。

## Client-Step2

1. 客户端根据前文公式生成`Salt`，`Iteration`及`Passowrd`计算出`SaltedPassword`。
2. 算出`Auth` = `UserIdentify` + `ClientNonce` + `Salt` + `MixNonce` + `MixNonce`。
3. 算出`SignedAuth` = HMAC(`Auth`, `HashedClientSignedPassword`)。
4. 算出`ClientProof` = `ClientSignedPassword` XOR `SignedAuth`。
5. 将`MixNonce`和`ClientProof`返回给服务器。

## Server-Step2

1. 算出`Auth` = `UserIdentify` + `ClientNonce` + `Salt` + `MixNonce` + `MixNonce`；注意第一个`MixNonce`应使用`Server-Step1`发出的，第二个`MixNonce`应使用`Client-Step2`发来的。
2. 算出`SignedAuth` = HMAC(`Auth`, `HashedClientSignedPassword`)。
3. 根据异或运算算出_客户端_的`ClientSignedPassword` = `ClientProof` XOR `SignedAuth`。
4. 算出`HashedClientSignedPassword` = Hash(`ClientSignedPassword`)。
5. 比对上一步算出的`HashedClientSignedPassword`与_数据库存储_的`HashedClientSignedPassword`是否一致，如果一致则说明密码_校验成功_。
6. 算出`ServerProof` = HMAC(`ServerSignedPassword`, `Auth`)。
7. 将`ServerProof`发给客户端。

## Client-Step3

1. 算出`ServerSignedPassword` = HMAC(`ServerPub`, `SaltedPassword`)。
2. 算出`ServerProof` = HMAC(`ServerSignedPassword`, `Auth`)。
3. 将`ServerProof`与_服务器_发来的`ServerProof`进行比较，如果一致则说明_校验成功_。