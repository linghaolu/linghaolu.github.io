---
layout: post
title:  "查看Android dex 方法数"
date:   2016-08-02
categories: android
tags: dex methods
---

# 65536 方法数 #

我们都知道 Android有一个 65536 (64k) 方法数限制，可以参考 [https://developer.android.com/studio/build/multidex.html](https://developer.android.com/studio/build/multidex.html)

但是这65536到底是哪些函数？只是自己class中的函数么？ 回答不是。

这 65536 个函数指的是函数引用，包含了自己类中的，以及引用的系统库函数。如果加起来超过65536，则会出现问题。

从上边连接中可以找到说明:
    
> The Dalvik Executable specification limits the total number of methods that can be referenced 
within a single DEX file to 65,536—including Android framework methods, library methods, 
and methods in your own code.
 
# 使用ClassyShark查看方法数 #

当然还有其他的工具，但是ClassyShark这个工具比较直观。[http://classyshark.com/](http://classyshark.com/)

具体可以查看 dex 总的方法引用数，以及自己class中实际的方法数。并且可以分package分别查看。

- 工具左边 classes 一栏，点击要查看的dex，显示出来的是 dex中所有引用的方法数，包括系统库函数。

    ![](/assets/posts/2016-08-02-dex-methods/dex-methods.png)

- 右边 Methods Counts 一栏，看到的是自己代码中的方法数，不包含引用的系统函数个数。

    ![](/assets/posts/2016-08-02-dex-methods/class-methods.png)
    

# Apk Analyzer #

[http://android-developers.blogspot.com/2016/05/android-studio-22-preview-new-ui.html](http://android-developers.blogspot.com/2016/05/android-studio-22-preview-new-ui.html)

Android studio 2.2 新引入的工具，可以查看方法数。

同时该工具可以查看 resources.arsc 文件。这个给力。

