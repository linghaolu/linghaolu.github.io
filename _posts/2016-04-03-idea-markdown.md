---
layout: post
title:  "Idea Markdown Support 插件"
date:   2016-04-03
categories: markdown
tags: idea markdown
---

# 强烈推荐 markdown + gitlab 作为文档管理系统 #

markdown 文档格式已经作为目前业内, 甚至产品界一个文档标准格式, 由于是纯文本, 编辑简单, 结合版本控制系统, 用起来那是刚刚的.

公司内有gitlab的,文档系统就用它,自由简单. 别的什么高级系统也别用了.

那就有个问题了,markdown 哪家编辑器比较强?

可能有很多选择, mac 下的 mou, windows 下的 markdown pad, 还有什么 atom, sublime 等.

因为工作原因, java 开发的越来越多的使用 Idea, 所以不管 mac linux windows 今天推荐 Idea 的官方插件 markdown support.
支持markdown基本语法,也支持常用的扩展table.

# markdown support 插件配置 #

虽然该插件支持表格,但是默认设置,还是不能正常工作,至少 mac 下不能正常工作. 需要手动设置一下, 把预览器设置为 JavaFx Webview 就能正常
显示表格了.

![setting](assets/posts/2016-04-02-idea-markdown/idea-markdown.png)