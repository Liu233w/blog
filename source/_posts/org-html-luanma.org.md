---
date: 2016/07/24 00:00:00
updated: 2018/03/25 22:48:19
tags:
- emacs
- org-mode
categories:
- blog
title: org 导出成 HTML 时在代码块末尾产生乱码的原因和解决方案
---

`fill-column-indicator.el` 是一个在代码的 80 列处自动显示一个竖线的插件， 如果我们把它的启动命令加到编程语言的 hook 中，org 导出的 HTML 会在 src block 中每一行的末尾出现乱码：

![img](16-7-24.PNG)

复制出来之后是这个 ``

我在 <https://github.com/alpaker/Fill-Column-Indicator/issues/45#issuecomment-108911964> 找到了解决办法，把下列代码复制进配置文件即可

```emacs-lisp
(defun fci-mode-override-advice (&rest args))
(advice-add 'org-html-fontify-code :around
            (lambda (fun &rest args)
              (advice-add 'fci-mode :override #'fci-mode-override-advice)
              (let ((result  (apply fun args)))
                (advice-remove 'fci-mode #'fci-mode-override-advice)
                result)))
```

EGO 生成的博客这下再也不会乱码了\_(:зゝ∠)\_
