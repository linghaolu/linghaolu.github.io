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

The bytecode verification process involves scanning through the instructions in every method in every class in a DEX file.
The goal is to identify illegal instruction sequences so that we don't have to check for them at run time. Many of the computations
involved are also necessary for "exact" garbage collection. See Dalvik Bytecode Verifier Notes for more information.

For performance reasons, the optimizer (described in the next section) assumes that the verifier has run successfully,
 and makes some potentially unsafe assumptions. By default, Dalvik insists upon verifying all classes, and only optimizes
  classes that have been verified. If you want to disable the verifier, you can use command-line flags to do so. See also
  Controlling the Embedded VM for instructions on controlling these features within the Android application framework.

Reporting of verification failures is a tricky issue. For example, calling a package-scope method on a class in a different package
is illegal and will be caught by the verifier. We don't necessarily want to report it during verification though -- we actually want to
throw an exception when the method call is attempted. Checking the access flags on every method call is expensive though.
The Dalvik Bytecode Verifier Notes document addresses this issue.

Classes that have been verified successfully have a flag set in the ODEX. They will not be re-verified when loaded.
The Linux access permissions are expected to prevent tampering; if you can get around those, installing faulty bytecode is
far from the easiest line of attack. The ODEX file has a 32-bit checksum, but that's chiefly present as a quick check for corrupted data.

Optimization

Virtual machine interpreters typically perform certain optimizations the first time a piece of code is used. Constant pool references are replaced with pointers to internal data structures, operations that always succeed or always work a certain way are replaced with simpler forms. Some of these require information only available at runtime, others can be inferred statically when certain assumptions are made.

The Dalvik optimizer does the following:

For virtual method calls, replace the method index with a vtable index.
For instance field get/put, replace the field index with a byte offset. Also, merge the boolean / byte / char / short variants into a single 32-bit form (less code in the interpreter means more room in the CPU I-cache).
Replace a handful of high-volume calls, like String.length(), with "inline" replacements. This skips the usual method call overhead, directly switching from the interpreter to a native implementation.
Prune empty methods. The simplest example is Object.<init>, which does nothing, but must be called whenever any object is allocated. The instruction is replaced with a new version that acts as a no-op unless a debugger is attached.
Append pre-computed data. For example, the VM wants to have a hash table for lookups on class name. Instead of computing this when the DEX file is loaded, we can compute it now, saving heap space and computation time in every VM where the DEX is loaded.
All of the instruction modifications involve replacing the opcode with one not defined by the Dalvik specification. This allows us to freely mix optimized and unoptimized instructions. The set of optimized instructions, and their exact representation, is tied closely to the VM version.

Most of the optimizations are obvious "wins". The use of raw indices and offsets not only allows us to execute more quickly, we can also skip the initial symbolic resolution. Pre-computation eats up disk space, and so must be done in moderation.

There are a couple of potential sources of trouble with these optimizations. First, vtable indices and byte offsets are subject to change if the VM is updated. Second, if a superclass is in a different DEX, and that other DEX is updated, we need to ensure that our optimized indices and offsets are updated as well. A similar but more subtle problem emerges when user-defined class loaders are employed: the class we actually call may not be the one we expected to call.

These problems are addressed with dependency lists and some limitations on what can be optimized.

Dependencies and Limitations

The optimized DEX file includes a list of dependencies on other DEX files, plus the CRC-32 and modification date from the originating classes.dex zip file entry. The dependency list includes the full path to the dalvik-cache file, and the file's SHA-1 signature. The timestamps of files on the device are unreliable and not used. The dependency area also includes the VM version number.

An optimized DEX is dependent upon all of the DEX files in the bootstrap class path. DEX files that are part of the bootstrap class path depend upon the DEX files that appeared earlier. To ensure that nothing outside the dependent DEX files is available, dexopt only loads the bootstrap classes. References to classes in other DEX files fail, which causes class loading and/or verification to fail, and classes with external dependencies are simply not optimized.

This means that splitting code out into many separate DEX files has a disadvantage: virtual method calls and instance field lookups between non-boot DEX files can't be optimized. Because verification is pass/fail with class granularity, no method in a class that has any reliance on classes in external DEX files can be optimized. This may be a bit heavy-handed, but it's the only way to guarantee that nothing breaks when individual pieces are updated.

Another negative consequence: any change to a bootstrap DEX will result in rejection of all optimized DEX files. This makes it hard to keep system updates small.

Despite our caution, there is still a possibility that a class in a DEX file loaded by a user-defined class loader could ask for a bootstrap class (say, String) and be given a different class with the same name. If a class in the DEX file being processed has the same name as a class in the bootstrap DEX files, the class will be flagged as ambiguous and references to it will not be resolved during verification / optimization. The class linking code in the VM does additional checks to plug another hole; see the verbose description in the VM sources for details (vm/oo/Class.c).

If one of the dependencies is updated, we need to re-verify and re-optimize the DEX file. If we can do a just-in-time dexopt invocation, this is easy. If we have to rely on the installer daemon, or the DEX was shipped only in ODEX, then the VM has to reject the DEX.

The output of dexopt is byte-swapped and struct-aligned for the host, and contains indices and offsets that are highly VM-specific (both version-wise and platform-wise). For this reason it's tricky to write a version of dexopt that runs on the desktop but generates output suitable for a particular device. The safest way to invoke it is on the target device, or on an emulator for that device.

Generated DEX

Some languages and frameworks rely on the ability to generate bytecode and execute it. The rather heavy dexopt verification and optimization model doesn't work well with that.

We intend to support this in a future release, but the exact method is to be determined. We may allow individual classes to be added or whole DEX files; may allow Java bytecode or Dalvik bytecode in instructions; may perform the usual set of optimizations, or use a separate interpreter that performs on-first-use optimizations directly on the bytecode (which won't be mapped read-only, since it's locally defined).

http://www.netmite.com/android/mydroid/dalvik/docs/dexopt.html