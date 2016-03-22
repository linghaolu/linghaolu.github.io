---
layout: post
title:  "Android N Jack 编译器负面影响"
date:   2016-03-22
categories: Android-N
tags: Android-N
---


Legacy javac toolchain:

	javac (.java --> .class) --> dx (.class --> .dex)

New Jack toolchain:

	Jack (.java --> .jack --> .dex)

由此可见，jack 不会产生 class文件，所以基于 class的lint等工具将无法工作。

包括依赖class的工具以及自己的编译脚本等。


参考

> http://developer.android.com/preview/j8-jack.html