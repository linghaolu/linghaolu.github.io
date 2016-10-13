---
layout: post
title:  "Tcl reopen redirected/closed stdout 重新打开stdout"
date:   2016-10-13
categories: tcl
tags: tcl
---

Tcl 中当 stdout 被从定向到文件或者 close 后，如果想重新回复到原来的命令行。   
代码如下



    C:\Users>tclsh
    % puts aaa
    aaa
    % close stdout
    set fd_log [open "a.txt" a]
    set stdout $fd_log
    puts aa
    close stdout
    open CON w
    stdout
    % puts aaa
    aaa

使用
   
    close stdout
    open CON w

重新打开，windows 使用 CON,  linux 使用 /dev/tty

