---
date: 2016/07/09 00:00:00
updated: 2018/03/25 22:48:19
tags:
- 前端
- emacs
- ego
categories:
- blog
title: 给博客换了个主题
---

- [以前的内容](#sec-)
- [16/7/10 修改](#sec-)
- [16/7/21 修改](#sec-)


# 以前的内容<a id="sec-"></a>

博客原来的主题是 EGO 自带的，直接拿来用的话重复度太高，所以我改了一下颜色\_(:зゝ∠)\_

其实我还改了一下头像的位置并且将站点中的一些英文改成了中文 ，原来的头像太靠下，会被一些文字挡住，现在我左移了搜索框， 上移了头像，舒服多了。看来 css 我还是会用一点的。

可惜现在博客又出现了新的问题，生成的程序代码会在每行的末尾出现乱码，我原来以为是 emacs 在 windows 系统上的问题，结果拿到 linux 上生成之后还是有问题。有可能是我修改了 org 的 配置文件导致的，目前还没有查出原因，真是神秘。

# 16/7/10 修改<a id="sec-"></a>

我已经找到乱码的原因了。是 fci 的锅。这个插件会在 buffer 的第 80 列位置显示一个竖线， 我把启动它的函数放到了 `prog-mode-hook` 里面，导致输出的时候调用了它，产生了乱码。 我本来打算在 ego 的输出调用函数里面（修改 `:ego-export-function` 参数）加上取消掉 hook 并且重置掉 fci，在输出结束之后再恢复 的代码，但是并不管用。所以我加入了一个列表，之后再列表里面的 hook 会被挂载上 fci 的启动函数， 同时所有的编程语言 buffer 仍然都会启动行号显示。

另外，我发现我的博客在手机端显示不正常，我的头像会挡住文字。于是我加了一个 js 判断语句， 可以在判断客户端是手机的情况下隐藏掉头像。

# 16/7/21 修改<a id="sec-"></a>

我发现这么做仍然没有什么卵用，输出的 c++代码没有乱码是因为 c-mode-hook 根本不会在 c++ mode 启动的时候执行，也就是说直接打开 c++的 buffer 也不会开启 fci mode，当然就不会有 乱码了。而 lisp 之类的仍然是有乱码的。我在 [这里](https://github.com/alpaker/Fill-Column-Indicator/issues/45#issuecomment-108911964) 找到了解决方案，只需要把一下的代码加到 config 里面就可以了：

```elisp
(defun fci-mode-override-advice (&rest args))
(advice-add 'org-html-fontify-code :around
            (lambda (fun &rest args)
              (advice-add 'fci-mode :override #'fci-mode-override-advice)
              (let ((result  (apply fun args)))
                (advice-remove 'fci-mode #'fci-mode-override-advice)
                result)))
```

现在即使把 `turn-on-fci-mode` 加到 `prog-mode-hook` 里面也不会有问题了。
