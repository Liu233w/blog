---
date: 2016/07/26 00:00:00
updated: 2018/03/25 22:48:19
tags:
- emacs
categories:
- blog
title: 在 windows 平台将 emacs 绿色化
---

- [概要](#sec-)
- [原理](#sec-)


# 概要<a id="sec-"></a>

其实在 windows 平台的 emacs 本来就是绿色软件，但是其配置文件的位置并不是可移动的，而是 用户的个人文件夹或者环境变量、注册表中的 home 变量指定的位置。

[这篇博客](http://blog.csdn.net/kuangdash/article/details/38298185) 里面有一个将配置文件放在 emacs 文件夹中的方法，但是这个代码只能将配置文件放在 `emacs 目录/share/emacs/当前版本号/` 之中，而我想要把含有配置文件的文件夹放在 emacs 根目录下，这是我的代码：

```emacs-lisp
(defvar program-dir
  (replace-regexp-in-string "share/emacs.*/etc/$" "home/" data-directory :from-end))
(setenv "HOME" program-dir)
(load "~/.emacs.d/init.el")
```

将上述代码放在 `emacs 目录/share/emacs/site-lisp/site-start.el` 中即可，没有这个文件就新建 一个。然后在 emacs 目录中新建一个 home 文件夹来存放配置文件，就可以在启动 emacs 的时候读取了。

# 原理<a id="sec-"></a>

`data-directory` 变量表示 emacs 文件夹下的 ect 文件夹地址，这里面含有 emacs 的地址。

```emacs-lisp
data-directory
```

    d:/TOTALCMD/tools/emacs/share/emacs/25.1/etc/

我们可以使用正则表达式将从 `share` 目录开始一直到字符串结尾的地址删除，这样就只保留了 emacs 目录，然后接上 home 目录就可以完整地表示配置文件的路径了。

```emacs-lisp
(defvar program-dir
  (replace-regexp-in-string "share/emacs.*/etc/$" "home/" data-directory :from-end))
program-dir
```

    d:/TOTALCMD/tools/emacs/home/

后面的代码将 emacs 中 HOME 环境变量设置成配置文件的路径，再读取配置文件 init.el，这样 emacs就完全的绿色化了。
