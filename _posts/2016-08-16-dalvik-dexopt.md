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

Application code is delivered to the system in a .jar or .apk file. These are really just .zip archives with some meta-data files added. The Dalvik DEX data file is always called classes.dex.

The bytecode cannot be memory-mapped and executed directly from the zip file, because the data is compressed and the start of the file is not guaranteed to be word-aligned. These problems could be addressed by storing classes.dex without compression and padding out the zip file, but that would increase the size of the package sent across the data network.

We need to extract classes.dex from the zip archive before we can use it. While we have the file available, we might as well perform some of the other actions (realignment, optimization, verification) described earlier. This raises a new question however: who is responsible for doing this, and where do we keep the output?

Preparation

There are at least three different ways to create a "prepared" DEX file, sometimes known as "ODEX" (for Optimized DEX):

The VM does it "just in time". The output goes into a special dalvik-cache directory. This works on the desktop and engineering-only device builds where the permissions on the dalvik-cache directory are not restricted. On production devices, this is not allowed.
The system installer does it when an application is first added. It has the privileges required to write to dalvik-cache.
The build system does it ahead of time. The relevant jar / apk files are present, but the classes.dex is stripped out. The optimized DEX is stored next to the original zip archive, not in dalvik-cache, and is part of the system image.
The dalvik-cache directory is more accurately $ANDROID_DATA/data/dalvik-cache. The files inside it have names derived from the full path of the source DEX. On the device the directory is owned by system / system and has 0771 permissions, and the optimized DEX files stored there are owned by system and the application's group, with 0644 permissions. DRM-locked applications will use 640 permissions to prevent other user applications from examining them. The bottom line is that you can read your own DEX file and those of most other applications, but you cannot create, modify, or remove them.

Preparation of the DEX file for the "just in time" and "system installer" approaches proceeds in three steps:

First, the dalvik-cache file is created. This must be done in a process with appropriate privileges, so for the "system installer" case this is done within installd, which runs as root.

Second, the classes.dex entry is extracted from the the zip archive. A small amount of space is left at the start of the file for the ODEX header.

Third, the file is memory-mapped for easy access and tweaked for use on the current system. This includes byte-swapping and structure realigning, but no meaningful changes to the DEX file. We also do some basic structure checks, such as ensuring that file offsets and data indices fall within valid ranges.

The build system uses a hairy process that involves starting the emulator, forcing just-in-time optimization of all relevant DEX files, and then extracting the results from dalvik-cache. The reasons for doing this, rather than using a tool that runs on the desktop, will become more apparent when the optimizations are explained.

Once the code is byte-swapped and aligned, we're ready to go. We append some pre-computed data, fill in the ODEX header at the start of the file, and start executing. (The header is filled in last, so that we don't try to use a partial file.) If we're interested in verification and optimization, however, we need to insert a step after the initial prep.

dexopt

We want to verify and optimize all of the classes in the DEX file. The easiest and safest way to do this is to load all of the classes into the VM and run through them. Anything that fails to load is simply not verified or optimized. Unfortunately, this can cause allocation of some resources that are difficult to release (e.g. loading of native shared libraries), so we don't want to do it in the same virtual machine that we're running applications in.

The solution is to invoke a program called dexopt, which is really just a back door into the VM. It performs an abbreviated VM initialization, loads zero or more DEX files from the bootstrap class path, and then sets about verifying and optimizing whatever it can from the target DEX. On completion, the process exits, freeing all resources.

It is possible for multiple VMs to want the same DEX file at the same time. File locking is used to ensure that dexopt is only run once.

Verification

The bytecode verification process involves scanning through the instructions in every method in every class in a DEX file. The goal is to identify illegal instruction sequences so that we don't have to check for them at run time. Many of the computations involved are also necessary for "exact" garbage collection. See Dalvik Bytecode Verifier Notes for more information.

For performance reasons, the optimizer (described in the next section) assumes that the verifier has run successfully, and makes some potentially unsafe assumptions. By default, Dalvik insists upon verifying all classes, and only optimizes classes that have been verified. If you want to disable the verifier, you can use command-line flags to do so. See also Controlling the Embedded VM for instructions on controlling these features within the Android application framework.

Reporting of verification failures is a tricky issue. For example, calling a package-scope method on a class in a different package is illegal and will be caught by the verifier. We don't necessarily want to report it during verification though -- we actually want to throw an exception when the method call is attempted. Checking the access flags on every method call is expensive though. The Dalvik Bytecode Verifier Notes document addresses this issue.

Classes that have been verified successfully have a flag set in the ODEX. They will not be re-verified when loaded. The Linux access permissions are expected to prevent tampering; if you can get around those, installing faulty bytecode is far from the easiest line of attack. The ODEX file has a 32-bit checksum, but that's chiefly present as a quick check for corrupted data.

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