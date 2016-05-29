---
layout: post
title:  "Git Submodule"
description: "Git Submodule"
date:   2016-05-28
categories: git
tags: git
---

submodule 其实类似于 maven ，主工程引用子工程的某个版本。

# 代码提交 #

先提交submodule 中的代码，然后再提交主程序中对于submodule的修改，主要是修改主程序对于submodule的引用。


# 代码更新 #

现在主工程中 git pull，然后再执行 git submodule update 才能把submodule的修改更新下来。

# 子模块分支的变动 #

子模块在本地分支的切换，在主工程中执行 git status / git diff 也会体现出来修改：

    iMac:submodule dj$ git status
    On branch master
    Your branch is up-to-date with 'origin/master'.
    Changes not staged for commit:
      (use "git add <file>..." to update what will be committed)
      (use "git checkout -- <file>..." to discard changes in working directory)

    	modified:   submodules/module1 (new commits)

    no changes added to commit (use "git add" and/or "git commit -a")

    iMac:submodule dj$ git diff
    diff --git a/submodules/module1 b/submodules/module1
    index 8eab28c..ff571fc 160000
    --- a/submodules/module1
    +++ b/submodules/module1
    @@ -1 +1 @@
    -Subproject commit 8eab28cfdf46192a3f13006385cb994b6ad15bb8
    +Subproject commit ff571fc6f05c3dec2d661bdf36f19a6dffe6eeba

这时候在主程序中提交，就可以把主程序对于子模块分支的引用提交到远程仓库。

---

参考 [https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97](https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)
