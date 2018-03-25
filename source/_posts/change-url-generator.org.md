---
date: 2016/08/22 00:00:00
updated: 2018/03/25 22:48:19
tags:
- EGO
- emacs
categories:
- blog
title: 使用 org 的文件名作为 EGO 生成的 URL
---

- [概述](#sec-)
- [改进方法](#sec-)
- [原理](#sec-)
- [16/8/23 更新](#sec-)
- [16/8/24 更新](#sec-)


# 概述<a id="sec-"></a>

EOG 默认是使用文章的标题来生成 URL 的，这么做的坏处有二：

1.  如果文章的标题里面有汉字，生成的 URL 因为编码的原因会变的特别长，不美观。
2.  只要更改了文章的标题，URL 就会随之而改变，以后就不好找了。而且 disqus 和多说也会把 修改标题之后的文章当成新的文章。

我修改了 EGO 生成 URL 的策略，现在我的博客就是使用 hack 之后的 EGO 来生成的。

# 改进方法<a id="sec-"></a>

```emacs-lisp
(defun liu233w//ego--generate-uri (default-uri-template creation-date title)
  "类似于`ego--generate-uri'，只不过读取 org 的文件名作为 title"
  (let ((uri-template (or (ego--read-org-option "URI")
                          default-uri-template))
        (date-list (split-string (if creation-date
                                     (ego--fix-timestamp-string creation-date)
                                   (format-time-string "%Y-%m-%d"))
                                 "-"))
        (encoded-title (ego--encode-string-to-url
                        ;; 获取 buffer 的文件名
                        (replace-regexp-in-string "^.*/\\|\\.org$" "" (buffer-file-name)))))
    (format-spec uri-template `((?y . ,(car date-list))
                                (?m . ,(cadr date-list))
                                (?d . ,(cl-caddr date-list))
                                (?t . ,encoded-title)))))
```

如果你不介意直接覆盖函数的话，可以把函数名换成 `ego--generate-uri` ，这样 EGO 的默认函数就 被覆盖了。如果你不想这么做，可以修改 `ego--category-config-alist` 中的 `:uri-generator` ， 如我的代码所示：

```emacs-lisp
(setf ego--category-config-alist
            '(("blog"
               :show-meta t
               :show-comment t
               ;; 将 org 的文件名当做 uri，参考函数
               :uri-generator liu233w//ego--generate-uri
               :uri-template "/blog/%y/%m/%d/%t/"
               :sort-by :date     ;; how to sort the posts
               :category-index t) ;; generate category index or not
              ("acm"
               :show-meta t
               :show-comment t
               ;; 将 org 的文件名当做 uri，参考函数
               :uri-generator liu233w//ego--generate-uri
               :uri-template "/acm/%y/%m/%d/%t/"
               :sort-by :date     ;; how to sort the posts
               :category-index t)
              ("index"
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
```

# 原理<a id="sec-"></a>

相比于原来的函数，我只是把 let 中的参数 `title` 换成了 `(replace-regexp-in-string "^.*/\\|\\.org$" "" (buffer-file-name))` 而已。因为这个函数会在每一个打开的 buffer 中运行，所以可以通过 `buffer-file-name` 来获取当前 org 文件的路径，然后用正则表达式把多余的内容删掉，只保留文件名即可。

表达式的效果：

```emacs-lisp
(replace-regexp-in-string "^.*/\\|\\.org$" ""
                          "d:/TOTALCMD/tools/emacs/home/documents/blog/blog/change-url-generator.org")
```

    change-url-generator

比较神奇的是 emacs 的正则表达式居然是把转义之后的 `|` 符号当做分支符号，而没有转义的 是符号本身，这跟其他的正则表达式实现不一样啊。

```emacs-lisp
(replace-regexp-in-string "ab|c" "dd" "ab|c")
```

    dd

```emacs-lisp
(replace-regexp-in-string "ab\\|c" "dd" "ab|c")
```

    dd|dd

貌似括号也是这样的。

# 16/8/23 更新<a id="sec-"></a>

作者根据[我的 issue](https://github.com/emacs-china/EGO/issues/85) 修改了这个函数，现在 EGO 可以在 URL 里面添加一个 `%f` 的模板来表示文件名 了。也就是说可以把 URL 改成 `/blog/%y/%m/%d/%f/` 来在生成的时候使用文件名作为 URI。但是由于 每一篇文章里面都有元数据的 URL，我就必须修改每一篇文章了。而且文章的模板没有相应的做出改变， `ego--insert-options-template` 和 `ego-new-post` 也没有改，目前使用这种方法还比较麻烦。 我暂时还不会修改自己的配置，等到这两个函数完善之后再说吧。

# 16/8/24 更新<a id="sec-"></a>

找到了新的办法：用 nadvice.el 修改 `ego--insert-options-template` 即可。请确保以下的 代码在 EGO 加载之后运行。

```emacs-lisp
(require 'nadvice)
;; 改变默认的元数据模板，使插入的 URL 为 "category/%y/%m/%d/%f/"
(defun org-config//advice-of-ego--insert-options-template (args)
  (setf (second args)
        (replace-regexp-in-string "%t/ Or .*$" "%f/"
                                  (second args)))
  args)

(advice-add 'ego--insert-options-template :filter-args
            'org-config//advice-of-ego--insert-options-template
            )
```

`nadvice.el` 是一个 emacs 内置的新 package，与 defadvice 类似，它可以改变 某个函数的默认行为。 `ego-new-post` 会调用 `ego--insert-options-template` 来插入 org 文件的元数据。 `advice-added` 函数通过参数 `:filter-args` 将我的自定义函数 套在了原来函数的外面，传递给原函数的参数要经过我的函数的处理才能真正传递给它。

如果你在新建文章的时候使用了默认生成的模板（调用 `ego-new-post` 函数的最后回答 yes）， 传递给 `ego--insert-options-template` 的第二个参数（URL 模板）默认会是 `/your-category/%y/%m/%d/%t/ Or /your-category/%t/` ，我的函数将通过正则表达式把它 替换成 `/your-category/%y/%m/%d/%f/` ，默认使用文件名当做 URL，这样就不需要再手动修改了。 如果你回答 no 来自己指定标题和 uri 的话，我的函数是不会影响的。

现在就可以去掉原来自定义的 `ego--generate-uri` 函数，把原来 org 文件里面的 `%t` 替换成 `%f` 了。上面的代码可比原来的修改少多了。

至于批量替换，可以用 emacs 的 dired 来做。用 dired 标记所有需要替换的文件，然后按 `Q` ， 就可以像在一个文件中替换文本那样替换所有文件中的文本了。
