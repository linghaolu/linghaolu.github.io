---
layout: post
title:  "proguard applymapping"
description: "Tcl使用cwind在Windows环境下UI自动化测试"
date:   2016-04-23
categories: tcl
tags: tcl cwind
---


# 背景 #

项目使用proguard混淆的时候，有时候需要不同的发布版本，混淆后的函数名要一样，比如多个模块之间的调用。
如果两个版本函数对不上的话，就找不到函数了。

虽然这种需求应该避免，但是场景还是有需求的。

# 解决办法 #

1、 找到上次proguard时生成的 mapping文件，命名为一个名字，比如 applymapping.txt。
2、 以后混淆，告诉proguard混淆时应用已有的mapping文件，保持已有函数混淆后的名字跟前一次对应。

> -applymapping applymapping.txt