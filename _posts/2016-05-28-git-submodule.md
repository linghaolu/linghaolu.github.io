---
layout: post
title:  "Git Submodule"
description: "Git submodule"
date:   2016-05-28
categories: submodule
tags: submodule
---

submodule 其实类似于 maven ，主工程引用子工程的某个版本。


代码提交：

先提交submodule 中的代码，然后再提交主程序中对于submodule的修改，主要是修改主程序对于submodule的引用。


代码更新：

现在主工程中 git pull，然后再执行 git submodule update 才能把submodule的修改更新下来。


参考 [https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97](https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)
