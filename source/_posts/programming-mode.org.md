---
date: 2016/08/21 00:00:00
updated: 2018/03/25 22:48:19
tags:
- emacs
- emacs package
categories:
- blog
title: 写了一个 emacs package：number-lock.el
---

- [项目简介](#sec-)
- [另一种实现方式](#sec-)
- [16/8/30 更新](#sec-)


# 项目简介<a id="sec-"></a>

项目地址：<https://github.com/Liu233w/number-lock.el>

这个 package 的主要功能与我的另一个脚本：[CapsLocKeys.ahk](https://github.com/Liu233w/CapsLocKeys.ahk) 中的一个功能类似， 可以把键盘上方的数字键替换成对应的符号，也就是说按下数字的时候会输入数字 上方的符号，而 `shift+数字` 则是数字本身。这个在写程序的时候非常有用。

但是那个脚本不能判断应用程序目前的状态，我只想在 evil 的 insert state 里面启用 替换的功能，而在其他模式底下关闭，而且在带有数字键的快捷键（比如 `C-x 0` ）里面 我也不想启用替换，所以只好写一个 emacs 的插件来替换按键了。

这个 package 定义了一个输入法：number-lock，只要用 `set-input-method` 把 输入法设置成这个，然后就可以用 `C-\` 来启用替换了。每个 buffer 的状态都是独立的， 并且只有在 buffer 的 insert-state 中才会开启，其他时候无论是在组合键还是在 minibuffer 中都不会启动。而且，如果有一些 package 在单个按键上绑定了函数，比如说 `paredit` 就在 `)` 上绑定了 `paredit-close-round` ，在按下右括号的时候可以把光标跳到最近 的右括号的外面，在使用了本 package 之后按下 `0` 之后可以调用这个函数，而不是仅仅 insert 一个 `)` 。

# 另一种实现方式<a id="sec-"></a>

我是不用 emacs 里面的输入法的，所以这种实现方式对我没有什么影响，但使用内置 输入法的同学可能会不方便一些。其实一开始我不是使用这种方法，而是使用 [Emacs-Wiki](https://www.emacswiki.org/emacs/Evil)上的方法 ，用 `key-translation-map` 来替换按键的，但是会产生一些 奇怪的 bug，[我把](https://github.com/Liu233w/programming-mode.el/blob/key-translate-mode/programming-mode.el)原来的代码保存在了另一个分支里面 ，如果有大神能帮我看看就 好了\_(:зゝ∠)\_

# 16/8/30 更新<a id="sec-"></a>

我的 package 已经正式登陆 melpa 了，只需要 `M-x package-install RET number-lock RET` 即可安装。

另外，melpa 上刚刚登陆了另一个和我的类似的插件： [evil-swap-keys](https://github.com/wbolster/evil-swap-keys) ，这个插件是用 `key-translation-map` 做到的，没有我之前实现中的 bug，并且很易于定制。 如果你还在 emacs 中使用别的输入法的话，这个 package 或许是一个不错的选择。
