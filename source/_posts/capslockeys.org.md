---
date: 2016/07/09 00:00:00
updated: 2018/03/25 22:48:19
tags:
- 脚本语言
- autohotkeys
categories:
- blog
title: 我的新项目：CapsLocKeys.ahk
---

我在 github 上发了一个新项目，一个 ahk 脚本，用来给大写锁定键增加新的功能。我受到了[这个链接](https://emacs-china.org/t/capslock/593/6?u=liu233w) 的启发，大写锁定键可以在点按和组合建的时候执行不同的操作。同时，我也提供了 一个数字键锁定的功能，在锁定状态下，按下主键盘上面的数字键将会输入符号， 而 shift+数字键才会输入数字。方便程序员编程。但是这样还是有缺陷，在 emacs 里面写程序的时候，我只希望在 insert 模式下打开数字键锁定，在其他模式下 关闭。下一步我打算编写一个 emacs 插件，在 emacs 上面实现数字键锁定的功能。

项目地址：<https://github.com/Liu233w/CapsLocKeys.ahk>
