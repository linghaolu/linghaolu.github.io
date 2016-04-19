---
layout: post
title:  "移动app https 接口使用自签名证书更合适"
date:   2016-03-18
categories: https
tags: https
---

# https 在移动端的应用 #

1. 防止web页面被劫持，篡改，插入代码等。
2. app 与服务端接口使用https增加数据安全性。


# https 存在的问题 #

https 也不是绝对安全的，存在中间人攻击。对于web页面https防止劫持这种，可以认为中间人攻击对他来说不起作用，因为想影响这么大范围，基本不可行。

对于app https 接口这种防止数据协议被分析这种出发点，可以认为https无法保证安全性。因为中间人攻击之时针对一个设备那还是很简单的。比如手机安装fiddler根证书，然后通过 fiddler上网，分分钟劫持，数据完全暴露。


# 如何解决 #

针对移动端的特殊性，app https 访问的域名可以使用自签名证书，有效期设置很长。app校验证书的时候自己控制，只认可这一个证书。这样就无法中间人的证书就不会生效。

# 注意 #

web 页面不能部署在这个自签名域名下，这个域名只能部署 api 接口。


参考

> http://www.oschina.net/translate/android-security-implementation-of-self-signed-ssl

这个链接中提到的demo其实代码其实不太合适。

更好的代码参考 google android 官方：
[http://developer.android.com/training/articles/security-ssl.html](http://developer.android.com/training/articles/security-ssl.html)