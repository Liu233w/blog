---
categories:
  - blog
date: 2018-04-02 11:47:24
tags:
- Vim
title: Vim 在插入模式下粘贴的方法
---

有时候我们需要在Vim的 command line （命令窗口、ex命令）进行粘贴，比如：
![](1.png)

在这个状态下我们显然不能用 `p` 来粘贴，而且在我的电脑上使用 `Ctrl+O` 也是不行的。

解决方法也很简单，使用 `Ctrl+R` 然后按寄存器名称就好了。比如 `Ctrl+R "` 就相当于 `p`，`Ctrl+R +` 就相当于 `"+p`。

参考： <https://stackoverflow.com/a/3997110>
