---
layout: post
title:  "Android Apk 动态写入数据方案，用于添加渠道号，数据倒流等"
date:   2016-04-02
categories: apk
tags: 渠道号 渠道包
---

# 背景 #

之前做过的项目中有几个功能用到了类似的功能。

- 打渠道包，每次发版需要出几十个上百个渠道包。
- 小说下载。用户在web页面点击下载阅读小说，第一步下载我们的阅读app，第二部用户打开该app后开始小说下载。
- apk高速下载。跟小说类似，这是个应用市场app，用户下载打开该应用市场后，在应用市场app中开始自动下载用户真正要下载的应用。

以上三个需求是具体经历过的，因为跨越年份较长，实现方式不大一样。

# 历史上的几种实现方式 #

- 往apk的res目录下放置一个配置文件。这种办法最简单，也最安全，别人无法篡改。但是缺点很严重，需要解压，再压缩，然后再签名。耗时较长，如果上百个渠道包，还是得有点耐心。另外就是app运行的时候读取也较慢。
- 往apk的meta-info目录下，通过 zip 命令 add 一个配置文件进去。这种办法效率也很高。但是app读取效率不是很高。需要初始化zip对象。
- 对于高速下载和小说这种，还有个很笨的办法，就是下载的时候根据内容的不同，通过浏览器下载的文件名也不同（通过http协议的Content-Disposition。然后启动的时候往浏览器下载目录下找到这个文件名。这种办法出错的概率太大。强烈不推荐。

以上几种方法其实一直都不是太满意。

还有另外一个办法略知一二。就是往 apk的最后追加数据。 但是听过 Android 5.0 以后就不能用了？

# 使用apk 的zip file comment 区域写入数据 #

上文中提到的往apk的后边追加数据，android 5.0 之前能用，之后不能用了。确实有此事，但是应该是用的方法不对。

之前的办法太鲁莽，直接往 apk 文件后边追加数据。但是 android 5.0 后开始校验 apk 数据格式合法性了。所以那种粗鲁的办法不能用了。那如何办？

我们先看下 apk（zip）文件的格式。因为 apk 本身是个 zip 格式, 格式可以参考[http://blog.sina.com.cn/s/blog_4c3591bd0100zzm6.html](http://blog.sina.com.cn/s/blog_4c3591bd0100zzm6.html).

对于这个格式我们不全看，只看最后的一个数据快。


|        | |End of central directory record | |
| ------------- |-------------| -----|----|
| Offset        | Bytes           | Description  |译|
| 0     | 4 | End of central directory signature = 0x06054b50 | 核心目录结束标记（0x06054b50）|
| 4      | 2      |  Number of this disk | 当前磁盘编号 |
| 6 | 2     | Disk where central directory starts | 核心目录开始位置的磁盘编号 |
| 8 | 2 | Number of central directory records on this disk | 该磁盘上所记录的核心目录数量 |
| 10 | 2 | Total number of central directory records | 该磁盘上所记录的核心目录数量 |
| 12 | 4 | Size of central directory (bytes) | 核心目录的大小 |
| 16 | 4 | Offset of start of central directory, relative to start of archive | 核心目录开始位置相对于archive开始的位移 |
| 20 | 2 | Comment length (n) | 注释长度 (n) |
| 22 | n | Comment| 注释内容 |

这个数据结构可以在 ZipOutputStream.java 的 finish 函数中参考：

    writeLeInt(ENDSIG);
    writeLeShort(0); /* disk number */
    writeLeShort(0); /* disk with start of central dir */
    writeLeShort(numEntries);
    writeLeShort(numEntries);
    writeLeInt(sizeEntries);
    writeLeInt(offset);
    writeLeShort(zipComment.length);
    out.write(zipComment);

我们需要关注的是最后两个字段，comment length 和 comment。

apk 默认情况下没有comment，所以 comment length的short 两个字节为 0，我们需要把这个值修改为我们的comment的长度，然后把comment追加到后边即可。

# 具体实现 #

这种办法生成效率极高，读取效率也是几种方法中最高的。非加密条件下10ms级别（nexus s）。

请参考项目: [https://github.com/linghaolu/apkcomment](https://github.com/linghaolu/apkcomment)
