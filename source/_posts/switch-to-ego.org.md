---
date: 2016/05/05 00:00:00
updated: 2018/03/25 22:48:19
tags:
- git
- 前端
- emacs
- ego
categories:
- blog
title: 成功将站点生成器转换为 EGO
---

- [以前的文章内容](#sec-)
- [16 年 5 月 9 日的修改](#sec-)
- [16 年 6 月 11 日修改](#sec-)


# 以前的文章内容<a id="sec-"></a>

把生成器换成 ego 了，人性化多了，比 org-page 不知道高到哪里去了。我更换生成器的起因 是博客突然没办法正常生成了，我以为是 org-page 和 spacemacs 起了冲突，结果后来发现并不是 ，是我的 git 在 emacs 上和 cmd 上的配置文件变得不是同一个了（我也不知道是怎么回事）， 于是每次用 emacs commit 的时候都会出错（要求用户指定用户名和邮箱），现在问题解决了， TMD 又浪费了一晚上。现在

我把配置文件中的 org-page layer 删掉了，改成了 githubblog layer。存放 ego 的配置文件。 另外，我找到了一个好东西：<http://elpa.popkit.org/> 这是 melpa 的国内镜像，用它下插件 方便多了。但是在 spacemacs 上修改源列表就麻烦了。因为源列表是内嵌在 core 里面的，如果 修改的话就会破坏 git 的版本控制，那就只好手动 fork 源文件然后自己保持同步了，简直蛋疼。 如果把修改的代码放到 layer 里面的话，在安装插件的时候会出错，因为 layer 是在 core 之后 加载的，但是插件会在 layer 之前安装，所以没什么卵用。在这里记录一下我的代码，缅怀我逝去 的半个小时

```lisp
  ;;将 melpa 源替换成 popkit 源，便于国内使用
(setq package-archives
      (delete-if '(lambda (it) (string= (car it) "melpa")) package-archives))
(add-to-list 'package-archives
             '("popkit" . "http://elpa.popkit.org/packages/"))

```

# 16 年 5 月 9 日的修改<a id="sec-"></a>

终于实现了我想要的效果，现在 blog 和 acm 的 category 已经显示在导航栏上了。 在 github 上给作者提问之后才知道怎么做的\_(:зゝ∠)\_

其实只需要在配置中加入如下代码：

```elisp
  (setf ego--category-config-alist
      '(("blog"
        :show-meta t
        :show-comment t
        :uri-generator ego--generate-uri
        :uri-template "/blog/%y/%m/%d/%t/"
        :sort-by :date     ;; how to sort the posts
        ;; 生成 category 的目录，默认是 nil，我改成 t 了，不知道原来是什么效果
        :category-index t)
       ("acm"
        :show-meta t
        :show-comment t
        :uri-generator ego--generate-uri
        :uri-template "/blog/%y/%m/%d/%t/"
        :sort-by :date     ;; how to sort the posts
        :category-index t)
       ("index"                         ;;底下的两个不能去掉，否则没法正常生成主页
        :show-meta nil
        :show-comment nil
        :uri-generator ego--generate-uri
        :uri-template "/"
        :sort-by :date
        :category-index nil)
       ("about"
        :show-meta nil
        :show-comment nil
        :uri-generator ego--generate-uri
        :uri-template "/about/"
        :sort-by :date
        :category-index nil)
       ))

;; 以下是重点：控制哪些 category 显示在导航栏中
(setf ego--category-show-list
      '("blog" "acm"))
```

现在还有一些可以改进的地方

-   使 category 在导航栏中的显示与其名称不同，比如把 blog 显示为“我的文章”
-   修改主题，现在的主题太难受了
-   使得 acm 和 blog 的 index 显示在主页上

# 16 年 6 月 11 日修改<a id="sec-"></a>

emacs-china 现在已经正式提供了 elpa 源，清华大学也已经提供了分流，包安装已经变得方便多了。

```emacs-lisp
;;清华大学的国内源：https://mirrors.tuna.tsinghua.edu.cn/help/elpa/
(setq configuration-layer--elpa-archives
    '(("melpa-cn" . "http://mirrors.tuna.tsinghua.edu.cn/elpa/melpa/")
      ("org-cn"   . "http://mirrors.tuna.tsinghua.edu.cn/elpa/org/")
      ("gnu-cn"   . "http://mirrors.tuna.tsinghua.edu.cn/elpa/gnu/")))
```

我在这个分流的地址上查到，如果想要修改 spacemacs 默认的源地址，需要在 user-init 中修改 configuration-layer--elpa-archives，spacemacs 提供了额外的一个变量 来保存源地址。现在已经不需要修改 spacemacs 的 core 了，所有的配置文件又保存到 同一个文件夹下了。
