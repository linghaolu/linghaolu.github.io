---
layout: post
title:  "Android N HTTPS"
date:   2016-03-25
categories: Android-N
tags: Android-N https
---

Android N 在 https 安全性，防止中间人攻击方面进行了加强。


# Default Trusted Certificate Authority #

app target api 设置为N，默认情况下，https 只能信任系统内置的证书，不再信任用户自己添加的证书。 这样我们平时使用的中间人攻击利器Fiddler就不能好好的工作了。

但是如果想使用自签名证书，或者其他指定证书，可以新添加的接口方便的配置，不再需要写代码，只是修改配置文件。（这个功能是要鼓励自签名证书啊）

自签名证书的相关信息参考另外一篇文章：[移动app https 接口使用自签名证书更合适](http://linghaolu.github.io/https/2016/03/18/mobile-app-https.html "移动app https 接口使用自签名证书更合适")


# Network Security Configuration #

Android N 新增了一个网络安全配置功能，只需要修改配置文件，不需要修改代码，就能够对指定的域名配置不同的证书策略：

- Custom trust anchors： 定制app信任的证书，比如设置信任指定的自签名证书或者指定信任某几个证书集合。
- Debug-only overrides： 在 debugable=true 测试环境下，增加对开发测试域名证书的信任。 
- Certificate pinning: 设置只信任几个特定证书。
- Cleartext traffic opt-out: 都是使用安全连接访问，避免http明文传输。


# Adding a Security Configuration File #

在 manifest meta-data添加配置文件。

	<?xml version="1.0" encoding="utf-8"?>
	...
	<app ...>
	    <meta-data android:name="android.security.net.config"
	               android:resource="@xml/network_security_config" />
	    ...
	</app>


# Customizing Trusted CAs #

为什么需要自定义信任证书呢，有如下几个原因：

- 使用自签名证书，防止中间人劫持
- 只信任内置CA的特定几个证书
- 信任非内置的证书。

# 设置使用自签名（非公开）证书 #

不再需要设那么多java代码，配置一下即可。

`res/xml/network_security_config.xml:`

	<?xml version="1.0" encoding="utf-8"?>
	<network-security-config>
	    <domain-config>
	        <domain includeSubdomains="true">example.com</domain>
	        <trust-anchors>
	            <certificates src="@raw/my_ca"/>
	        </trust-anchors>
	    </domain-config>
	</network-security-config>

把自签名或者非公CA证书，以PEM 或者DER 格式放在 res/raw/my_ca

# 设置信任的特定几个[根]证书 #

我们不想信任所有内置的CA证书（root后安装根证书？）

`res/xml/network_security_config.xml:`

	<?xml version="1.0" encoding="utf-8"?>
	<network-security-config>
	    <domain-config>
	        <domain includeSubdomains="true">secure.example.com</domain>
	        <domain includeSubdomains="true">cdn.example.com</domain>
	        <trust-anchors>
	            <certificates src="@raw/trusted_roots"/>
	        </trust-anchors>
	    </domain-config>
	</network-security-config>

certificates 可以设置多个

# Trusting Additional CAs #

设置系统的 & 额外的证书

`res/xml/network_security_config.xml:`

	<?xml version="1.0" encoding="utf-8"?>
	<network-security-config>
	    <base-config>
	        <trust-anchors>
	            <certificates src="@raw/extracas"/>
	            <certificates src="system"/>
	        </trust-anchors>
	    </base-config>
	</network-security-config>

# Configuring CAs for Debugging #

当我们开发测试阶段，app 链接到本地部署的 server，该server没有产品最终域名部署的证书。方便调试，当 debugable = true 情况下通过`debug-overrides` 设置调试 CA.

`res/xml/network_security_config.xml:`

	<?xml version="1.0" encoding="utf-8"?>
	<network-security-config>
	    <debug-overrides>
	        <trust-anchors>
	            <certificates src="@raw/debug_cas"/>
	        </trust-anchors>
	    </debug-overrides>
	</network-security-config>

# Pinning Certificates #

默认app值信任内置的CA根证书。如果其中又被污染的则会遭到中间人攻击。 可以通过设置有限集合的方式来保证安全。

Certificate pinning 通过设置证书公钥的哈希值来限制的。只有证书链上包含这个hash的公钥，这个证书就有效。

但是我们会更换证书，或者更换CA机构，所以这种情况下我们需要设置一个备份hash。 并且可以设置有效期，有效期过后，此配置不再生效。

`res/xml/network_security_config.xml:`

	<?xml version="1.0" encoding="utf-8"?>
	<network-security-config>
	    <domain-config>
	        <domain includeSubdomains="true">example.com</domain>
	        <pin-set expiration="2018-01-01">
	            <pin digest="SHA-256">7HIpactkIAq2Y49orFOOQKurWxmmSFZhBCoQYcRhJ3Y=</pin>
	            <!-- backup pin -->
	            <pin digest="SHA-256">fwza0LRMXouZHRC8Ei+4PyuldPDcf3UKgO/04cDM1oE=</pin>
	    </domain-config>
	</network-security-config>

---------------
> 原文 ： [http://developer.android.com/preview/features/security-config.html](http://developer.android.com/preview/features/security-config.html)