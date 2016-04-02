---
layout: post
title:  "使用openssl 生成RSA pem格式密钥对"
date:   2016-04-02
categories: openssl
tags: openssl rsa pem
---

使用linux， macox， windows cygwin 环境的openssl 生成rsa非对称加密的pem格式密钥对。


1、
`openssl genrsa -out rsa_private_key.pem 1024`

该命令会生成1024位的私钥,此时我们就可以在当前路径下看到rsa_private_key.pem文件了.

2、
生成的密钥不是pcs8格式，我们需要转成pkcs8格式。

`openssl pkcs8 -topk8 -inform PEM -in rsa_private_key.pem -outform PEM -nocrypt`

3、
生成 rsa 公钥

`openssl rsa -in rsa_private_key.pem -pubout -out rsa_public_key.pem`





