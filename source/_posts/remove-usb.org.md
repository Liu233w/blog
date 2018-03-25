---
date: 2016/09/19 00:00:00
updated: 2018/03/25 22:48:19
tags:
- usb
- ahk
- windows
categories:
- blog
title: 在 windows 下使用快捷键安全快速地移除 USB 设备
---

- [序言](#sec-)
- [简介](#sec-)
- [一个脚本](#sec-)


# 序言<a id="sec-"></a>

默认情况下，在 windows 系统中想要移除 USB 设备(比如 U 盘)只有一种方法：用鼠标单击系统托盘 中的图标，选择你想要移除的设备。但是用鼠标的效率很低，而如果想要使用键盘快捷键来移除 USB 设备，就只能使用某些命令行程序或者用 powershell 或 ahk 脚本来调用系统 dll 来做到了。

但是有一些命令行工具和脚本并不安全，我想要的效果是如果 U 盘被占用，就不要弹出这个设备。 然而很多脚本会强制弹出设备。最后，我找到了一个比较好用的命令行工具，[RemoveDrive](http://www.uwe-sieber.de/drivetools_e.html)。

# 简介<a id="sec-"></a>

这个工具的使用非常简单： `RemoveDrive.exe d:` 即可弹出盘符为 d 的 usb 设备。这个工具 在弹出设备之前会首先尝试接触占用，如果失败，则不会弹出设备，并且打印出异常信息。是我 目前见到过的最好的此类工具。

# 一个脚本<a id="sec-"></a>

为了便于使用快捷键弹出设备，我写了一个 ahk 脚本。把它跟上文的 exe 文件放在一起， 启动时会列出电脑上所有的可移除设备，只需要在键盘上敲出盘符（不需要冒号）然后按下回车 即可自动调用上述的 exe 来弹出设备。现在只需要给这个脚本分配一个快捷键就好了。

```ahk
;确定可弹出的驱动器列表
DriveList = Empty
DriveGet, DriveList, List, REMOVABLE

if DriveList
    goto, start_eject

MsgBox, 没有需要弹出的驱动器
return

start_eject:

InputBox, Driveletter, 请输入要弹出的盘符 , %DriveList%
if ErrorLevel=1
    return

StringUpper, Driveletter, Driveletter ;将用户输入的盘符转换成大写

IfInString, Driveletter, %DriveList%
{
    Driveletter = %Driveletter%:
    Runwait, RemoveDrive.exe %Driveletter%

}
else
{
    Msgbox, 请输入一个列表中的盘符
    goto, start_eject
}
```
