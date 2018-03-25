---
date: 2017/05/12 00:00:00
updated: 2018/03/25 22:48:19
tags:
- windows
- wsl
categories:
- blog
title: WSL(Windows Subsystem for Linux) 提示找不到某程序的可能原因
---

今天我久违的打开了 WSL，结果发现它报了一个奇怪的错误：

    PS C:\WINDOWS\system32> bash
    bash.exe"-3.1$ apt-get --version
    bash.exe": apt-get: command not found
    bash.exe"-3.1$

其实这是开错了程序导致的。我也安装了 Git for windows，并且把 `c:\Program Files (x86)\Git\bin\` 路径加进了环境变量里面（为了能在命令行打开 git）。然而这个路径中 也包含了 `bash.exe` ，所以其实是开成这个程序了。把这个文件删掉或者改成别的名字就 好了。
