---
layout: post
title:  "Python main 函数"
date:   2016-07-30
categories: python
tags: python
---

一个python模块代码无论是被导入还是被直接执行，模块中的代码都会运行。如何判断该模块是被导入还是被直接执行呢？

答案是判断 `__nanme__` 系统变量。

- 如果模块是被导入，__name__ 的值为模块名字.
_ 如果模块是被直接执行，__name__ 的值为 '__main__'

所以在模块中经常有这样的语句:

    if __name__ == '__main__' :
        test()
        

这里放置测试代码是个不错的好地方。
