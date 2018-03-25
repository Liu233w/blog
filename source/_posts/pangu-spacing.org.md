---
date: 2016/07/24 00:00:00
updated: 2018/03/25 22:48:19
tags:
- spacemacs
- emacs
categories:
- blog
title: Spacemacs 通过选择性开启 pangu-spacing 加快打开大文件的速度
---

- [概述](#sec-)
- [实验](#sec-)
- [解决方法](#sec-)


# 概述<a id="sec-"></a>

spacemacs 的 Chinese layer 中 pangu-spacing 是默认全局开启的，这个 package 会严重拖慢 emacs 打开大文件或者在大文件中翻动页面的速度。如果使用类似 [子龙山人的配置](https://github.com/zilongshanren/spacemacs-private/blob/develop/layers/zilongshanren-better-defaults/config.el#L132%5D%5D%0Ahttps://github.com/zilongshanren/spacemacs-private/blob/develop/layers/zilongshanren-better-defaults/config.el#L132) 里面的在打开文件时检测大小并关闭行号和 pangu-spacing 的函数，在打开大文件的时候仍然 极其卡顿，因为 emacs 会先调用 pangu-spacing 添加空格的函数，然后再调用 `find-file-hook` 来关闭 pangu-spacing。

# 实验<a id="sec-"></a>

```emacs-lisp
  (add-hook 'find-file-hook '(lambda ()
                             ;;模拟遇到大文件时关闭 pangu-spacing
                             (pangu-spacing-mode -1)
                             (message "find-file-mode-hook")))
(add-hook 'pangu-spacing-mode-hook '(lambda () (message "pangu-spacing-mode")))
```

message 中打开一个文件时的输出：

<p class="verse">
find-file-mode-hook<br />
pangu-spacing-mode [4 times]<br />
</p>

虽然不知道为什么调用了好几次，但是 buffer 里面 pangu-spacing 确实是关闭的。 这么做在打开大文件时仍然极其卡顿。

# 解决方法<a id="sec-"></a>

覆盖 spacemacs 对 pangu-spacing 的默认配置。默认使其关闭，只有当我们确定 buffer 很小的时候才 启动它。这个函数是我直接从 chinese-layer 里面复制的，只改动了一部分和加了一个 hook。

```emacs-lisp
  ;; 多大的 buffer 才算是大 buffer，在 buffer-size 超过此大小之后会采取不同的策略
;; 来提高运行效率
(defvar *large-buffer-threshold* 300000
  "Buffer whose size beyond it will have a different behavior for the efficiency")

(defun liu233w/init-pangu-spacing ()
  "覆盖 Chinese-layer 中的设置。默认关闭 pangu-spacing，只有在 buffer 比较小的时候才启动，
如果是启动之后再关闭的话就开的太慢了。"
  (use-package pangu-spacing
    :config
    (global-pangu-spacing-mode -1)
    (spacemacs|hide-lighter pangu-spacing-mode)
    ;; Always insert `real' space in org-mode.
    (add-hook 'org-mode-hook
              '(lambda ()
                 (set (make-local-variable 'pangu-spacing-real-insert-separtor) t)))

    (defun liu233w/enable-pangu-spacing-when-buffer-not-large ()
      "when buffer is not large, turn on it"
      (when (< (buffer-size) *large-buffer-threshold*)
        (pangu-spacing-mode 1)))

    (dolist (i '(prog-mode-hook text-mode-hook))
      (add-hook i 'liu233w/enable-pangu-spacing-when-buffer-not-large))
    ))
```

这么做之后打开大文件比原来流畅多了。

我原来试过把配置放进 `post-init-pangu-spacing` 里面，但是不成功，默认pangu-spacing还是 开启的， 只好直接覆盖函数了。附上我原来的配置，希望大家能帮我找一下哪里有问题\_(:зゝ∠)\_

```emacs-lisp
(defun liu233w/post-init-pangu-spacing ()
  (with-eval-after-load 'pangu-spacing-mode
    (pangu-spacing-mode -1)

    (defun liu233w/enable-pangu-spacing-when-buffer-not-large ()
      "when buffer is not large, turn on it"
      (when (< (buffer-size) *large-buffer-threshold*)
        (pangu-spacing-mode 1)))

    (dolist (i '(prog-mode-hook text-mode-hook))
      (add-hook i 'liu233w/enable-pangu-spacing-when-buffer-not-large))))
```
