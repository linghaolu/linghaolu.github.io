---
layout: post
title:  "Android N Hehavior Changes"
date:   2016-03-24
categories: Android-N
tags: Android-N
---

# Background Optimizations #

- target api 设置为N后，android.net.conn.CONNECTIVITY_CHANGE 注册在manifest中将不再生效。但是如果通过BroadcastReceiver代码注册的，并且当app运行在前台的时候，才可以工作。

- android.hardware.action.NEW_PICTURE android.hardware.action.NEW_VIDEO 两个 broadcast 将被禁止使用。

	> 以上两点可以通过 JobScheduler 进行替代方案。[http://developer.android.com/preview/features/background-optimization.html](http://developer.android.com/preview/features/background-optimization.html)

- Android N 应该会出来限制app后台运行的机制，禁止app依赖于后台运行的service这种工作方法。并且给出了新的手段，让开发者在这种状态下测试app的兼容性，并且建议开发者移出这些功能。！！！以后push咋搞！！！。

	- To simulate conditions where implicit broadcasts and background services are unavailable, enter the following command:	  
	
 		`$ adb shell cmd appops set RUN_IN_BACKGROUND ignore`    


	- To re-enable implicit broadcasts and background services, enter the following command:   
	
		`$ adb shell cmd appops set RUN_IN_BACKGROUND allow`   


# Screen Zoom #

用户可以设置屏幕density，开发者需要保证软件在 sw320dp 宽度下可以正常显示。

# NDK开发改动 #

ndk 开发只能调用 public api，如果调用非public api 在将来的官方release版本中则会导致crash。目前只是在logcat中给出error提示，提醒开发者尽快修改。

如果编译时依赖了系统的 library，比如 libpng，但是不属于ndk开放的部分，则apk需要把该so打包到自己apk中。如果使用了第三方的so，他很可能使用了非public api，所以一定要确定检查确认。





参考

> [http://developer.android.com/preview/j8-jack.html](http://developer.android.com/preview/behavior-changes.html)