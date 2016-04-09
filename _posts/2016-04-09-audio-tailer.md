---
layout: post
title:  "Android平台Amr语音边录边发"
description: "Android平台Amr语音边录边发,参考Apache common io tailer实现."
date:   2016-04-09
categories: amr
tags: amr
---

# 技术背景 #

类似于微信语音消息,边录制边发送,在慢速网络下,能够较好的提高用户体验,录制完也就发送完.

# 技术分析 #

Android平台提供了两种语音录制接口, MediaRecorder 和 AudioRecorder.

## MediaRecorder ##

MediaRecorder 可以直接输出语音到文件,格式为 amr,简单使用的话,就是录制完毕后,把语音amr文件,再发送. 而不能简单的边录制便获取数据.

## AudioRecorder ##

AudioRecorder 提供的接口不错,边录制边返回数据,符合我们的要求,但是数据格式为pcm原始数据, 数据量很大, 并不能直接用来直接发送.

网上也有不少文章介绍如何把 pcm 转为 amr, 另外系统也有一个类(非public) AmrInputStream, 介绍用它来把pcm转为amr,
但是这种非公开类风险比较大,用起来也比较费劲. 强烈不推荐.


# 最优实现方案 #

我们想找的还是一个简单可以来的方法,简单稳定.

还是采用MediaRecorder 直接输出amr到文件. 我们调研发现, mediarecorder 录制完毕后,对 amr 文件没有重填,也就是对于文件头也不会
做进一步处理. 就是一边录制一边往后边一帧一帧的添加数据, 录制结束,文件也就结束.

其实这里我们也可以参考amr文件格式, 具体的参考[http://blog.csdn.net/dinggo/article/details/1966444](http://blog.csdn.net/dinggo/article/details/1966444)

我们简单看一下:

| 文件头 | 语音帧1 | 语音帧2 | 语音帧...|

其实是个流式格式.

这里我们就想到了 Linux 的 tailer命令, 以及 apache common io 中的 tailer. 用途一样, 监控文件变化,文件有增加, 就输出, 一边变化一边输出.

网上还有通过MediaRecorder接口把文件直接输出到 localsocket 流中,然后边录制边读取, 这种方案略微复杂,并且 android 5.0 由于限制无法工作.

我们要做的就是把 apache common io tailer 文件修改一下,就能很好的满足我们的需求.
[https://commons.apache.org/proper/commons-io/apidocs/src-html/org/apache/commons/io/input/Tailer.html](https://commons.apache.org/proper/commons-io/apidocs/src-html/org/apache/commons/io/input/Tailer.html)


具体项目实现参考: [https://github.com/linghaolu/tailer](https://github.com/linghaolu/tailer)