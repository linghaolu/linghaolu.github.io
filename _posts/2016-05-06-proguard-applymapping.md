---
layout: post
title:  "proguard applymapping"
description: "Tcl使用cwind在Windows环境下UI自动化测试"
date:   2016-05-06
categories: proguard
tags: proguard
---


# 背景 #

项目使用proguard混淆的时候，有时候需要不同的发布版本，混淆后的函数名要一样，比如多个模块之间的调用。
如果两个版本函数对不上的话，就找不到函数了。

虽然这种需求应该避免，但是场景还是有需求的。

# 解决办法 #

1. 找到上次proguard时生成的 mapping文件，命名为一个名字，比如 applymapping.txt。
2. 以后混淆，告诉proguard混淆时应用已有的mapping文件，保持已有函数混淆后的名字跟前一次对应。

在配置文件中添加如下:

> -applymapping applymapping.txt

# 发现的问题 #

云华大诗人在项目中应用过程中遇到点问题，提示warning：

> Warning: ... is not being kept as ..., but remapped to ...

Proguard 官网对这个也有解释，但是不属于那个原因。

最后发现一个配置去掉后就没问题了

> -dontshrink

就是他，去掉后就没有报告那个warning了，还是挺诡异的。不过这个配置确实最好还是没有的好，shrink 过程会去掉没有用的class等，减小包体积。
