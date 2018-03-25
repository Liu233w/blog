---
date: 2016/06/16 00:00:00
updated: 2018/03/25 22:48:19
tags:
- clang-format
- emacs
- LLVM
- c++
categories:
- blog
title: 在 emacs 中使用 clang-format 自动格式化 c++代码
---

- [序言](#sec-)
- [需要的工具](#sec-)
- [安装步骤](#sec-)
- [需要改进的方面](#sec-)
- [16/11/3 更新](#sec-)
  - [正文](#sec-)
  - [编码问题](#sec-)
  - [TIPS](#sec-)


# 序言<a id="sec-"></a>

Visual Studio 中的 c++ 编辑器有一个很强大的功能，就是在键入分号或者右大括号之后可以自动格式化当前区域的代码。这个功能也可以在 Emacs 中实现。

# 需要的工具<a id="sec-"></a>

-   emacs（废话）
-   clang-format（windows 版的 clang 中自带了，linux 版需要自己安装）
-   evil（我的函数使用了 evil 中的一些函数） -- EDIT：我最新的代码取消了这个限制
-   spacemacs（可选，只是配置起来方便一些）

# 安装步骤<a id="sec-"></a>

这里使用 spacemacs 做示范，在.spacemacs.el 或者.spacemacs.d/init.el 的 layer 函数中找到 c++ layer，把它替换成

```emacs-lisp
(c-c++ :variables
       c-c++-enable-clang-support t
       )
```

如果没有找到 c++的话直接加上这一行就可以了。对于不使用 spacemacs 的用户，可以参考[官方网站](http://clang.llvm.org/docs/ClangFormat.html)。

然后我们需要这两个函数来做格式化：

```emacs-lisp
(defun liu233w/semi-clang-format (args)
  "format by clang-format when enter ';'"
  (interactive "*P")
  (c-electric-semi&comma args)
  (clang-format-region (line-beginning-position 0) (line-beginning-position 2))
  )

(defun liu233w/brace-clang-format (args)
  "format by clang-format when enter '}'"
  (interactive "*P")
  (c-electric-brace args)
  (let ((end-position (point))
        begin-position)
    (save-excursion
      (evil-jump-item)
      (setf begin-position (point)))
    (clang-format-region begin-position end-position))
  )
```

linux 用户注意：Ubuntu 版 clang-format 的可执行文件名不是 clang-format，而是 clang-format-<版本号>，在使用这两个函数之前需要做一个 链接：

```shell
cd /usr/bin
sudo ln clang-format-<版本号> clang-format
```

然后加上快捷键：

```emacs-lisp
;;add clang-format support
(add-hook 'c++-mode-hook
          (lambda ()
            (when (executable-find "clang-format")
              ;;使用 clang-format 作为默认排版工具
              (local-set-key (kbd "C-M-\\") 'clang-format)
              ;;当插入分号时自动对当前行排版
              (local-set-key (kbd ";")
                             'liu233w/semi-clang-format)
              (local-set-key (kbd "}")
                             'liu233w/brace-clang-format))))
```

我的代码只能在 c++-mode 下使用，现在只要输入分号（如果使用 evil 的话要在插入模式下输入）就可以自动对包括当前行和前后两行在内的代码格式化。 如果输入右大括号就会跳到最近的右大括号之后，并且对整个代码块进行格式化。和 Visual Studio 不同的是不能通过删除右大括号再重新插入的方式来格式化 代码块——这样代码就乱了（这是 c-electric-brace 函数的原因）。不过有了右大括号跳到代码块之外的功能和括号自动补全之后也就不需要单独输入右大括号了吧。

clang-format 还有一个格式化配置文件的功能。可以把一个命名为.clang-format 的文本文件放在源代码的同级或上级目录中来控制 clang-format 的格式化行为。 具体配置方法请参考[http://clang.llvm.org/docs/ClangFormatStyleOptions.html](http://clang.llvm.org/docs/ClangFormatStyleOptions.html)。 这里贴上我的配置文件

```shell
BasedOnStyle: google
IndentWidth: 4
BreakBeforeBraces: Allman
AllowShortFunctionsOnASingleLine: false
```

# 需要改进的方面<a id="sec-"></a>

clang-format 的格式化行为缺少默认值，如果能在找不到配置文件的时候使用一个默认的就好了。等我把 elisp 学好之后再加上这个功能吧。

# 16/11/3 更新<a id="sec-"></a>

## 正文<a id="sec-"></a>

今天完成了新版的代码，改动如下：

-   独立成了一个单独的 package，虽然目前还是只能用在 c 和 c++ 上，但是我懒得改了\_(:зゝ∠)\_
-   在没有提供配置文件的时候自动提供一个默认的配置选项，此选项可以定制
-   去除了对 evil 的依赖。

新的代码在：<https://github.com/Liu233w/.spacemacs.d/blob/master/layers/liu233w/local/auto-clang-format/auto-clang-format.el>

使用方法： `(add-hook 'c++-mode-hook #'acf-enable-auto-clang-format)` spacemacs 用户可以参考[我的配置。](https://github.com/Liu233w/.spacemacs.d/commit/5040d6f32c7b91811f40256e39d1830aae2a3bfc)

## 编码问题<a id="sec-"></a>

最近更新了新版的 emacs25 和 spacemacs 0.200.1，发现 clang-format 没法正常格式化代码了 （windows 系统）。因为 clang-format 程序会使用 windows 下的 CR LF 行尾，而 emacs 保存 的临时文件是 Unix 的 LF 行尾。使用下面的代码可以修复（在 clang-format 之后求值）：

```emacs-lisp
(when (spacemacs/system-is-mswindows)
  (defun liu233w//utf-8-dos-as-coding-system-for-write (func &rest rest)
    (let ((coding-system-for-write 'utf-8-dos))
      (apply func rest)))
  (advice-add #'clang-format-region :around
              #'liu233w//utf-8-dos-as-coding-system-for-write))
```

## TIPS<a id="sec-"></a>

-   locate-dominating-file 函数可以在指定的文件（第一个参数）的当前目录和上级目录中寻找指定的 文件（夹）名（第二个参数）。找到时返回其绝对地址，找不到返回 nil。
