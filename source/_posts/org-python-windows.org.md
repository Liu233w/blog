---
date: 2016/09/29 00:00:00
updated: 2018/03/25 22:48:19
tags:
- org
- emacs
- windows
categories:
- blog
title: 解决 Emacs 的 Python 交互执行环境在 windows 下无法显示中文的问题
---

- [问题描述](#sec-)
- [解决方案](#sec-)
- [org-babel 的问题](#sec-)
- [TIPS](#sec-)
- [16/9/30 更新](#sec-)
- [16/10/12 更新](#sec-)
- [16/12/5 更新](#sec-)


# 问题描述<a id="sec-"></a>

我把 Emacs 默认的编码改成了 utf-8，代码为：

```emacs-lisp
(set-language-environment "chinese-GBK")
(prefer-coding-system 'utf-8)
```

由于 windows 中的 shell 默认只支持中文的 gbk 编码，强制修改全部的编码会导致乱码，所以用这种 方法可以得到一个“prefer”的编码系统，在 shell 里面就不会乱码了。但是如果在 shell-mode 下的 python 的交互环境中输入中文却会得到异常，并且交互编程时显示的中文会变成像 `\320\320\315\352\261\317` 这样的编码形式。

# 解决方案<a id="sec-"></a>

本来这就是改一下编码就能解决的问题，理论上可以把上面代码第二行的 utf-8 改成 `gbk` 就可以了。 但是我为了保持文件的兼容性，没法采用这样的配置。只好把 terminal 的编码单独改成 gbk，在 上面的代码下面加上这几行即可：

```emacs-lisp
(when (spacemacs/system-is-mswindows)
  ;; from http://paxinla.github.io/2015/07/12/Windows%E4%B8%8BEmacs%E7%9A%84shell-mode%E4%B9%B1%E7%A0%81%E8%A7%A3%E5%86%B3/
  ;; 解决 Shell Mode(cmd) 下中文乱码问题
  (set-terminal-coding-system 'gbk)
  (modify-coding-system-alist 'process "*" 'gbk)
  (defun liu233w/windows-shell-mode-coding ()
    (set-buffer-file-coding-system 'gbk)
    (set-buffer-process-coding-system 'gbk 'gbk))
  (add-hook 'shell-mode-hook #'liu233w/windows-shell-mode-coding)
  (add-hook 'inferior-python-mode-hook #'liu233w/windows-shell-mode-coding)
  )
```

注意那个 `inferior-python-mode-hook` ，如果你还有别的交互环境出现了这种问题，可以把相应的 hook 加上这个函数。

这下无论是交互环境还是 shell 都不会出问题了。

# org-babel 的问题<a id="sec-"></a>

由于我们将进程编码改的与文件编码不同了，使用 org-babel 的时候返回的内容会在行末多出来一个 `^M` ，我们还需要用别的代码来处理这个问题。这里的 `remove-dos-eol` 是我从子龙山人的配置里 抄过来的。

```emacs-lisp
(defun liu233w/remove-dos-eol ()
  "Replace DOS eolns CR LF with Unix eolns CR"
  (interactive)
  (goto-char (point-min))
  (while (search-forward "\r" nil t) (replace-match "")))

;; 如果是 windows，则自动处理 org-babel-eval 的返回值，防止使用 gbk 编码的
;; cmd 返回值行末符号不匹配
(with-eval-after-load 'ob-python
  (defun liu233w/fix-org-babel-execute:prog (string)
    (with-temp-buffer
      ;; (message (format "debug %s" string))
      (insert string)
      (liu233w/remove-dos-eol)
      (buffer-string)))
  (advice-add #'org-babel-execute:python :filter-return
              #'liu233w/fix-org-babel-execute:prog))
```

# TIPS<a id="sec-"></a>

-   org-bable 为每种语言单独创建的文件中有一个 `org-babel-execute:语言名` 函数， 用来执行当前的代码块，这个函数返回一个字符串，即执行的结果。
-   `with-temp-buffer` 宏可以创建一个临时的 buffer，在执行完 body 之后会将其删除。
-   `advice-add` 函数的 `:filter-return` 标签可以将某个函数的返回值在返回调用处之 前交给指定的函数处理。

# 16/9/30 更新<a id="sec-"></a>

今天发现如果需要使用 yapf 的话还需要再加一个 advice

```emacs-lisp
(advice-add #'py-yapf-buffer :after #'liu233w/remove-dos-eol)
```

# 16/10/12 更新<a id="sec-"></a>

发现了更加精简的方法，把下面的代码放进 config 中即可：

```emacs-lisp
(set-language-environment "UTF-8")
(prefer-coding-system 'utf-8)
(when (spacemacs/system-is-mswindows)
  (set-language-environment "chinese-gbk")
  (set-default 'process-coding-system-alist
               '(("[pP][lL][iI][nN][kK]" gbk-dos . gbk-dos)
                 ("[cC][mM][dD][pP][rR][oO][xX][yY]" gbk-dos . gbk-dos)))
  (defun liu233w//python-encode-in-org-babel-execute (func body params)
    "org-babel 执行代码时不会自动编码文件，这里通过动态作用域覆盖默认选项来编码文件。"
    ;; 此问题的详细信息请参考：https://github.com/Liu233w/.spacemacs.d/issues/6
    (let ((coding-system-for-write 'utf-8))
      (funcall func body params)))
  (advice-add #'org-babel-execute:python :around
              #'liu233w//python-encode-in-org-babel-execute))
```

这段代码的说明参见：<https://github.com/Liu233w/.spacemacs.d/issues/6> 今后如果还有 babel 的语言出现类似的问题的话，只需要用 `function-add` 来用这个函数修改 对应的 `org-babel-execute` 函数即可。

TIPS:

-   `edebug` 可以 debug elisp, 参见 <https://www.gnu.org/software/emacs/manual/html_node/elisp/Edebug.html>

# 16/12/5 更新<a id="sec-"></a>

其实上面的代码还是有问题的，今天终于把这个问题彻底解决了，代码如下：

```emacs-lisp
;; 设置编码
(cond
 ((eq system-type 'windows-nt)          ; 或者 (spacemacs/system-is-mswindows)
  ;;
  (set-language-environment "chinese-gbk")
  (prefer-coding-system 'utf-8)
  (set-terminal-coding-system 'gbk)
  ;;
  (modify-coding-system-alist 'process "*" 'gbk)
  (defun liu233w/windows-shell-mode-coding ()
    (set-buffer-file-coding-system 'gbk)
    (set-buffer-process-coding-system 'gbk 'gbk))
  (add-hook 'shell-mode-hook #'liu233w/windows-shell-mode-coding)
  (add-hook 'inferior-python-mode-hook #'liu233w/windows-shell-mode-coding)
  ;;
  (defun liu233w//python-encode-in-org-babel-execute (func body params)
    "org-babel 执行代码时不会自动编码文件，这里通过动态作用域覆
盖默认选项来编码文件。"
    ;; 此问题的详细信息请参考：https://github.com/Liu233w/.spacemacs.d/issues/6
    (let ((coding-system-for-write 'utf-8))
      (funcall func body params)))
  (advice-add #'org-babel-execute:python :around
              #'liu233w//python-encode-in-org-babel-execute))
 ;; --
 (t
  (set-language-environment "UTF-8")
  (prefer-coding-system 'utf-8)))
```

1.  `set-language-environment` 和 `prefer-coding-system` 的部分设置会互相覆盖，所 以顺序很重要。
2.  经试验发现（因为我也不知道原理），这里的修改 process 部分的编码和之前的那个 [简化版](#org3f91627) 产生的效果还是不一样的，这段代码更好一些。
3.  不再需要 `remove-dos-eol` 了。
4.  不再需要修改 yapf 的设置了。
5.  不会导致别的地方产生乱码（至少我测试到的是这样）

我所有的修改记录都保存在 <https://github.com/Liu233w/.spacemacs.d/issues/6> 和 <https://github.com/Liu233w/.spacemacs.d/issues/8> 里面了。
