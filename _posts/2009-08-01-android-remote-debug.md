---
layout: post
title:  "使用Eclipse远程调试Android"
date:   2009-08-01
categories: debug
tags: Android Debug
---

# 问题背景 #

我们使用eclipse调试开发的apk没有问题，但是如果需要调试framework、service、RIL等中间层的代码如何调试呢？下面介绍一下使用Eclipse结合DDMS使用java的远程调试方法来调试Android中间层代码。

# 调试方法 #

这里我们以调试RIL代码为例，其他代码方法类似。

1. 创建RIL相关代码的Eclipse工程

	选择Eclipse菜单: File -> New -> Java Project  (不选择Android project)
	
	选择 Create project from existing source, 然后选择要调试代码的目录。（当然我们也可以选择Android根目录，把所有代码都添加进来）

    ![](/assets/posts/2009-08-01-android-remote-debug/newproject.png)


	出现编译错误我们不用管，也不需要解决。就这样就可以了。

	![](/assets/posts/2009-08-01-android-remote-debug/warning.png)

2. 在DDMS中选择我们要调试的进程

	这一步很重要，调试之前一定要先选择要调试的进程，DDMS才能知道我们要调试哪个,这里我们选择 com.android.phone

3. 配置debug remote java application

	菜单 Run -> Debug configrations

	![](/assets/posts/2009-08-01-android-remote-debug/debug.png)

	双击Remote Java Application，注意 connection properties选择本机，端口8700（具体是什么可以通过ddms配置）。这个端口是DDMS工具为了调试开放的端口。


OK下面你就可以设置断点，调试任何想调试的代码了。