---
layout: post
title:  "Dalvik Optimization and Verification With dexopt"
description: "Dalvik Optimization and Verification With dexopt"
date:   2016-08-16
categories: dalvik
tags: dalvik dexopt optimization verification
---

#  #


Dalvik 虚拟机是专门针对Android这种移动平台设计的. 移动平台一般内存较小, 磁盘读取也很慢.

针对这些特点和限制必须主要聚焦以下目标:

- Class 数据尤其是类的字节码,必须多进程共享, 最小化内存使用.
- 启动app的开销要尽可能的小,这样手机才能相应的过来.
- 如果像传统java一样没个类存储成一个独立的文件, 会有很多数据重复, 尤其是字符串. 为了节省磁盘空间, 需要想办法解决这个问题.
- 解析类的 feild 时候会有额外的加载class的开销. 最好加载这些 value (比如 integer string等) 能够像c语言那样直接访问.
- 字节码的 verification 很有必要, 但是这个校验过程很慢, 所以需要在程序执行前就把这个 verification 过程执行完.
- 对字节码进行optimization优化处理是很有必要的, 这样能够提升执行速度和节省电量.
- 出于安全考虑,程序间不能修改共享的代码.


传统的经典java虚拟机运行类的时候先从压缩的文件中解压出来,然后存储在内存中. 这种实现方式, 没个进程中都回有一份独立的拷贝, 并且程序的启动速度
也会比较慢, 因为需要一个解压过程(或者从磁盘上读取一个一个的独立的class数据). 另外在本地内存中存储这些字节码,似的重新改写这些指令变得很方便,
这样促使进行一些优化操作.

这些目标让我们做出以下决定:

- 所有的 class 文件合并成一个 "DEX" 文件.
- DEX 文件是只读的,并且进程间共享使用.
- 字节的顺序和字节对齐方式根据操作系统进行调整过的.
- 字节码的 verification 必须对所有类都要强制执行, 但是最好是提前进行.
- Optimizations 对字节码的修改也需要AOT, 程序执行前进行.

分以下几个部分进行讲解.

# VM Operation #

应用程序代码是通过jar或者apk的方式交给系统运行的. jar 和 apk 其实就是 zip 文档, 额外的加了一些 meta-data 元数据. Dalvik DEX 数据文件
叫做 classes.dex.


字节码并不能够直接从zip文件map到内存直接运行, 因为数据是压缩的, 并且无法保证字节对齐. 这个问题可以通过不对 dex 文件压缩, 活着干脆把dex从
zip中拿出来来解决. 但是这样回增加安装包的体积.


所以我们需要在程序真正执行前把 classes.dex 从zip中解压出来. 同时我们还可以做一些上边提到的对齐, 优化, 校验等工作. 这就涉及到谁负责做
这些事情, 处理后的输出文件放在哪里?

## Preparation ##

至少有三种方式创建这种预处理后的 DEX 文件, 也就是我们所说的 ODEX( Optimized DEX):

- 虚拟机通过 "Just in time" JIT 方式. 输出文件到 dalvik-cache 目录. 这种方式在工程机可以, 但是真正的release后的设备无法运行.
因为系统的 dalvik-cache 目录有访问权限限制, 普通的app 没有权限.
- 系统的 installer 在app安装后执行. installer 有权限写如 dalvik-cache.
- 编译系统的时候通过 "ahead of time" AOT 的方式. 把 dex 从 apk 中解压出来, 并且执行优化等操作,生成odex文件, 但是不放到/data/Dalvik-cache
 目录, 而是放到 /system/app 目录下.


dalvik-cache 目录是在 /data/dalvik-cache 目录下. 文件权限 0771, 属于 system. app 只能读取不能修改.

对于 "just in time" 和 "system installer" 方式生成 odex 文件有以下三个步骤:

- 第一, 一般通过 system installer (installd)来创建 dalvik-cache 目录.
- 第二, 把 classes.dex 从 zip文件中解压出来, 并且在文件开始部分预留部分区域, 目的是写入 ODEX 头.
- 第三, odex文件可以mmap到内存直接快速访问, 并且对当前运行的系统做一些调整. 包括 byte-swapping 和 structure realigning等. 但是不会
对原dex文件做什么修改. 同时会做一些基本的structure校验, 比如保证文件偏移和数据索引不越界等.

之所以不再电脑上提前把这些事情处理好, 而在运行时处理是有一定原因的, 后边会进行讲解. (这个对app开发的有一定影响, 因为 dexopt 执行耗时较长
, 但是也无法提前进行dexopt, 只能运行在目标机器上进行处理)

当进行过 byte-swap 和 align 后, 会填充odex 头. 然后就可以开始执行程序. 如果关心 verification 和  optimizaiton, 执行前需要增加一个dexopt步骤.
(dexopt其实是必须要执行的)

## dexopt ##

我们要求对dex中的所有class进行verify和optimize. 最简单的方式是把所有的class加载到虚拟机并且执行一遍. 如果有失败则verify-optimize
过程失败. 但是很不幸, 这么做会分配太多的资源并且很难释放(比如加载过的native lib), 所以这个校验过程不能和我们最后运行app虚拟机进程是同一个.

解决的办事是调用一个叫做 dexopt 的程序, 运行在一个独立进程(运行时是fork一个进程出来), 其实是个虚拟机的小后门程序.
dexopt 直至行一些小规模的vm初始化, 并且加载 dex, 然后执行 verify 和 optimize 工作. 当完成后退出进程, 释放所有资源.

当多个vm需要对同一个文件进行 opt 时, 需要一个文件锁进行同步处理, 该文件只有一个dexopt进程对他进行处理.

# Verification #

字节码校验过程需要扫描所有类的所有方法指令. 目标是在这个阶段识别所有非法指令, 而不用在运行时来做检查.


出于性能考虑, 下一章要讲的optimizer假设verifier成功执行了. 并且会有些潜在不安全的假设. 默认情况下 dalvik 会校验所有class, 并且只对
校验过的class 做 optimize. 如果想disable verifier 可以通过命令行 flag 来操作. 另外可以通过
[http://www.netmite.com/android/mydroid/dalvik/docs/embedded-vm-control.html](http://www.netmite.com/android/mydroid/dalvik/docs/embedded-vm-control.html)
如果在framework 层控制.


报告 verification 错误是个比较复杂的事情, 比如包访问权限违规, 我们没必要在娇艳阶段报告错误,实际上是在运行时报告. 这种检查还是比较费时的.
可以通过 [http://www.netmite.com/android/mydroid/dalvik/docs/verifier.html](http://www.netmite.com/android/mydroid/dalvik/docs/verifier.html)
更详细的了解.


被校验通过的 class 会在 ODEX 文件中做个标记 flag. 加载的时候不会再次校验.

# Optimization #

虚拟机解释器通常在代码第一次运行时做一些优化工作. 比如常量池的引用会转化为内部数据结构指针的方式, 对于经常成功执行活着某种工作方式的代码,
会转化成另外一种更简单的形式. 其中一些工作只能在运行时进行, 另外一些可以通过一些规则静态的来进行.


Dalvik optimizer 做以下事情:

- 对于 virtual method 调用, 使用 vtable 虚函数指针列表索引 代替 文件中的 method 索引.
- 对于对象的 field get/put 操作, 用 1 byte offset 来代替 field 索引. 另外合并 boolean / byte / char / short 成一个 32 bit 形式.
(less code in the interpreter means more room in the CPU I-cache).
- 把一些经常调用的简单函数比如 String.length() 改为内联函数. 这样就减少函数调用的消耗.
- 修剪空方法. 最简单的例子 Object.<init>, 什么也不做. 但是每次对象分配时都得必须调用. 指令被替换成另外一种新的实现, 除非有 debugger
attach 上了, 要不然什么也不做.
- 添加一些可以提前计算好的数据. 比如 对于虚拟机查找 class name 的 hashtable. 可以提前计算好, 而不需要执行的时候再计算.


所有的这些指令修改涉及到替换一些不受虚拟机规范约束的 opcode 替换. 这样可以更自由的对 optimized 和 unoptimized 指令进行组合. 优化过的
指令和他们所代表的实际意义跟虚拟机版本绑定.


大部分优化工作是成功的. 使用原始的索引和偏移不仅仅可以执行的更快, 并且可以忽略符号表的初始化. 提前计算很吃磁盘空间, 所以这个提前计算工作
需要适度.


优化工作还是有潜在的一些麻烦的. 第一 vtable 虚函数表索引 和 字节 offset 是会发生变化的如果虚拟机更新的话. 第二 如果一个class的superclass
在另外一个dex文件中, 并且那个dex更新了, 那么我们就需要保证我们这个dex需要重新更新索引和偏移(也就是需要重新进行opt). 一个类似的更微妙
的情景, 当我们自定义一个classloader 时候, 调用一个class 有可能实际不是我们想要的class.(指的是第一个dex中的类通过自定义classloader加载
另一个dex中的class ?)

These problems are addressed with dependency lists and some limitations on what can be optimized.


# Dependencies and Limitations #

optimize后的 odex file 包含了一个对其他dex 的一来列表, 同时包含odex对应的原始 classes.dex 文件的修改时间和crc校验码.  依赖列表包含
dalvik-cache 文件的全路径, 已经sha-1签名. 时间戳不使用的系统文件的时间戳, 应为不可信(使用的应该是zip文件中的时间字段). 依赖数据同时包含vm的版本号.

optimized dex 依赖于所有的系统bootstrap class path 中的 dex 文件. bootstrap 中的越是依赖于其他bootstrap 中的dex的越靠前.
为了保证除了所依赖的dex文件以外所有其他都不可用, dexopt 只是加载 bootstrap 中的class. 依赖其他 dex 中的class 会导致load 和 verificatin 错误,
有依赖外部dex的class 就简单的没有被optimized.

这就意味着当把代码打包成多个dex的时候有个负面的影响: 自己的多个dex间的方法调用或者变量引用没有办法 optimized. 因为 verification 过程是
类颗粒度的, 没有办法对依赖外部dex的类进行optimize. 虽然这么做有点过分, 但是这是唯一办法可以保证单独的一个dex有更新, 不影响其他dex.


另外一个负面影: 对于系统bootstrap dex 的更新, 回到之依赖这些的优化过后的 dex 失效,需要重新进行opt.


尽管我们很小心,但还是有可能自定义一个classloader使用自定义一样名字的类来返回系统的类,比如 String. 在dex 中, 如果一个class的名字和bootstrap
dex 中的一样. 则这个class 被标记为有歧义的, 并且在optimize和verify过程中不能被识别. The class linking code in the VM does additional checks to plug another hole;
see the verbose description in the VM sources for details (vm/oo/Class.c).


dexopt 输出的 odex 文件是相对于其运行的设备 byte-swapped, struct-aligned. 并且包含的索引 偏移量等是高度和虚拟机吻合的(包括版本,平台等).
出于这个原因, 不大可能写一个pc上跑的一个 dexopt 来提前完成这个工作. dexopt 最安全的办法还是在运行的设备上执行, 或者对应的模拟器上.


----------
以上文字仅仅是个翻译, 是最近再看相关资料时感觉有必要整理下.

原文链接[http://www.netmite.com/android/mydroid/dalvik/docs/dexopt.html](http://www.netmite.com/android/mydroid/dalvik/docs/dexopt.html)
