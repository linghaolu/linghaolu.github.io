---
layout: post
title:  "Android app security"
date:   2016-02-17
categories: Security
tags: Android Security
---

> 以下文字为某次听见做记录

# 数据泄漏 #
- 本地文件敏感数据不能明文保存，不能伪加密（Base64，自定义算法等）
- android:allowbackup=false. 防止 adb backup 导出数据
- Activity intent 的数据泄漏。比如通过 getRecentTask  然后找到对应的intent 拿到数据。
- Broadcast Intent，自己应用内使用 LocaBroadcast，避免被别的应用收到，或者 setPackage做限制。
- ClipBorad 数据泄漏。
- WebView settings setSavePassword(false)  这个会明文保存密码。
- Log 要关闭，防止重要数据泄漏。使用boolean常量开关 或者 proguard 直接优化。
- 键盘事件的读取预防，/dev/input/event 可以读取到按键和触屏。键盘建议随机布局自定义。
- 对于截屏 android5.0以及以后 window.setFlag(LayoutParam.FLAG_SECURE) 禁止录屏。

# 客户端暴露的攻击面 #
- 使用外部数据不进行校验，比如app升级，插件安装等，需要对这些数据进行合法性校验。
- zip解压目录覆盖风险。zip中允许 ../../file 这样的路径。如果一旦解压到当前目录，有可能覆盖上级目录的文件。
- Android components 不当暴露，不需要export的需要 exported = false。
- 本地端口开放问题，socket server。尽量不要开放此接口。如果要的话，也只是bind到 127.0.0.1，不要暴露给局域网，避免局域网内恶意代码扫描端口。另外 app 可以通过 读取proc 通过 端口查看是哪个app连接该端口。（收selinux限制）。另外就是使用此方法不要实现一些特别敏感的功能。
- 开放组件的 dos攻击。开放的 activity service receiver等需要对传入的intent做合法性校验，以及相关的类型转换保护。防止恶意代码攻击。
- PendingIntent，不要给第三方app发送PendingIntent。避免数据被修改。

# 界面的劫持 #
- 恶意悬浮框，在我们的app上边覆盖一个悬浮框，误导用户点击不合理的按键。这时候需要设置setFilterTouchesWhenObscured 为 false，被别的窗口覆盖的话，不接受按键。
- 钓鱼窗口，当用户打开我们的界面时，恶意程序也打开一个类似的钓鱼界面。我们需要在关键的界面 onPause的时候 做必要的检查，比如看看栈顶是不是自己的界面。（5.0以后受限制？）
- ContentProvider sql 注入。参数中包含恶意sql;-- 。最简单的是做 sql参数校验。
- ContentProvider openFile 便利目录风险。

# Webview远程方法调用漏洞 #
- 4.2以下手机 addJavaScriptInterface 会导致漏洞。js通过 getClass 后获取java 类，然后调用相关函数。系统自己带一个 searchbox_xxx 需要自己移出掉。

# 不安全的网络通信 #
- 中间人攻击。
- 敏感数据不要明文传输。
- 恶意wifi可以通过 kali linux 很简单的创建。在商场进行钓鱼。
- 加密算法
	- RC4 已经过时，不推荐使用。
	- SHA256 最好，不推荐md5 sha1
	- RSA 要 2048 bit，要 padding。
	- 对称加密密钥不要放在代码中。可以协商后保存在本地加密存储。
	- AES 不要使用 ECB 模式，初始化向量不要使用固定的常量。
	- SecureRandome 不要使用setSeed()，使用也不要传入固定值
- https 中间人攻击
	- cookie 要设置为 secure（secure flag），否则该cookie会在 http会话中传输。
	- 不要使用 SSLv3以及更低版本
	- 在程序中不要自己处理证书相关的校验。

- SSL证书校验
	- webview onReceivedSslError后 不要自己做什么处理。
	- android 系统中有时候某些手机证书不全，但是也不能忽略该证书错误。
	- 不要覆盖 Trustmanager. checkServerTrusted 不要重写。
	- HostNameVerifier 不要重写。不要不校验 hostname。
- 如何处理呢？
	- 通过TrustManagerFactory 导入证书。
	- 证书绑定。就是我只认这个证书。自己做 veriry。成本最低。证书可以是自签名的。
# 二进制攻击 #
- 各种黑产QQ群论坛等，看雪论坛。
	- 重新打包，插入恶意代码
	- 逆向分析
	- 运行时debug，修改数据等
- 工具
	- apktool，dex2jar, JEB
	- IDA pro （查看 so 代码，F5 汇编转c代码）
	- xposed，Cydia substrate 注入框架
- 防护
	- 理论上没有100%有效的地域二进制攻击的方法。
	- 但是为啥还要这么做呢？提高门槛，提高成本，提高他的利益成本（有这时间他可以去找些软柿子赚钱去）
	- proguard做混淆
- 安全性校验
	- 检查apk有没有被修改
	- 检查签名（不靠谱，此处代码可以被修改）。但是有比没有强。黑产都是些批量自动化的，可以防止一些。
	- 增加难度
		- 放在 native代码中
		- 多点检查
		- 检查的代码不要放在退出点，放在比较隐蔽的地方。然后后边别的地方再退出程序。
		- 和网络请求结合，传参数到server，server返回不合法数据等。
- 反调试，反注入
	- debuggable = false
	- Debug.isDebuggerConnected 进行检查。
	- 监控 JDWP 线程（hook socket，进行数据过滤）
	- 多进程ptrace保护。进程只能被ptrace一次。（多个进程间需要pipe通信监控ptrace进程是否退出，监听到，主程序也退出）
	- 检查tracerPid，被trace后为不为0（也可以被绕过）
	- 检查 gdb android_server gdbserver 是否在手机上（可改名）
	- 检查 xposed框架是否在运行。
	- 检查是否被 hook（java，GOT， inline）
	- 检查设备是否被root或者是在 emulator上运行。
	- 检查 jailbreak（iOS）
- 字符串混淆加密
	- java native中的字符串都要做混淆。代码放在 native 层。
	- 隐藏native层的函数名， dlsym
	- obfuscator-llvm 混淆 natived代码。支持 SUB FLA BCF 等几种模式。
- 其他native保护
	- so 中检查签名
	- jni函数名混淆
	- 删除所有不需要 export 的符号。编译选项中设置。
	- elf tricks，设置一些数据让工具 crash。
	- so整体加密。加壳。开源的 upx。
	- 特定函数加密。
- 应用加固
	- 非定制化方案，无混淆，无字符串加密。
	- hook系统代码，等，有比较大的兼容性问题。
	- 影响启动速度。
	- 无 so 层保护。