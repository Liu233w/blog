---
date: 2016/11/15 00:00:00
updated: 2018/03/25 22:48:19
tags:
- emacs
categories:
- blog
title: windows 下用来快速启动 emacs client 的脚本
---

我一直在 windows 下使用 emacs 的 client 模式。但是有一件事情让我非常的不爽，就是有时候 emacs server 在关闭之后不会自动清除掉 server 文件，导致下次打开的时候也不会用新的 server 文件覆盖原来的，令 client 程序无法正常开启。于是我写了下面这个脚本，可以在开 启 emacs 之前强制删除 server 文件：

```bat
set serverPath=home\.emacs.d\server\server
set executePath=bin\runemacs.exe

del %~dp0%serverPath%
start /b %~dp0%executePath% %1
```

请确保路径正确，我将脚本放在了 emacs 根目录，就是有 bin 之类文件夹的地方。而 home 目 录也在这个路径的 home 文件夹下。脚本命名为 `remove-server-file-and-run.cmd` 。 `start` 命令可以让 emacs 启动的时候不会保留命令行的黑框。

还有另一个脚本可以让 client 无法启动的时候运行上面的脚本来创建一个 server：

```bat
set serverPath=home\.emacs.d\server\server
set executePath=bin\emacsclientw.exe
set alternativeEditor=remove-server-file-and-run.cmd
start /b %~dp0%executePath% --server-file %~dp0%serverPath% %1 -a %~dp0%alternativeEditor%
```

这个脚本放在与刚才的脚本相同的目录中，命名为 `run-client-or-start-server.cmd` 。 同刚才一样，请确保路径正确。

这样，我们就可以使用 `run-client-or-start-server.cmd 文件名` 来打开对应的文件了。 如果服务器不存在或 server 文件失效，脚本会自动删除 server 文件并开启一个新的 emacs 来 打开当前文件。如果你在 emacs 的配置中进行了一些设置的话，emacs 的启动会开启一个新的 server。

比如在 total commander 中可以将上面的脚本绑定到一个快捷键上，然后一键用 emacs 打开文件。。。
