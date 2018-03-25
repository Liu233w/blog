---
date: 2016/09/02 00:00:00
updated: 2018/03/25 22:48:19
tags:
- spacemacs
- emacs
categories:
- blog
title: 在 spacemacs 生成 micro-state 的同时为每个命令生成对应的执行并进入 micro-state 的函数
---

- [需求](#sec-)
- [我的实现](#sec-)
- [使用](#sec-)
  - [另一个宏](#sec-)
  - [示例 1](#sec-)
  - [示例 2](#sec-)
- [经验](#sec-)


# 需求<a id="sec-"></a>

spacemacs 有一个叫做 micro-state 的功能，宏 `spacemacs|define-micro-state` 可以生成 一个 micro-state 和相应的函数。当调用函数进入 micro-state 的时候，可以暂时应用一个 临时的键绑定。比如定义一个翻页的 micro-state，启动之后可以用 d u f b 来翻页，不需要 按下 Ctrl。如果按下这四个按钮之外的按键，就会退出 micro-state 并执行那个按键原本的 功能。

这个 micro-state 还有另一个功能：可以指定一个在进入 micro-state 的时候求值的表达式。 比如定义 micro-state 为四个翻页方式，设定在进入 micro-state 的同时求值 `(evil-scroll-page-down 0)` ，然后把激活 micro-state 的函数绑定到 `C-f` 上。 这样在按键翻页的时候只有第一次需要按 `C-f` ，之后只需要按 `f` 来翻页就可以了。

但是这样做只能指定一个动作，如果我想在按下 `C-b` 的时候向上翻页并进入 micro-state， 这就没有办法了。而且对于刚才的设置，我们是没法使用 `2 C-f` 这样的操作来向下翻两页的。 在 [emacs-china 的一个帖子](https://emacs-china.org/t/ctrl/1039/20?u=liu233w) 中提供了一个思路，就是定义一个函数，这个函数拥有 interactive 和参数，会先调用原来的函数，然后激活 micro-state。

# 我的实现<a id="sec-"></a>

我定义了一个宏，可以自动完成上面所说的操作。代码在： <https://github.com/Liu233w/.spacemacs.d/blob/master/libs/multiple-micro-state.el>

宏的名字是 `mms|define-multiple-micro-state` ，使用方式与 `spacemacs|define-micro-state` 相同。注意这个只能在 spacemacs 中使用，因为它使用了宏 `spacemacs|define-micro-state` 。

在执行之后，会按照 `spacemacs|define-micro-state` 的命名方式生成一个 micro-state， 并且给它定义了一个别名 `名称:micro-state` ，并且给定义中的每一个函数生成一个名为 `名称:函数名-then-enter-micro-state` 的函数，这个函数不接受参数，有一个不接受参数的 interactive 形式。调用之后会使用 `command-execute` 来执行原来的函数。

同时，如果有一些使用 `evil-define-command` 定义的没法获取 interactive 命令的函数的话， 这个函数会自动加上一个 interactive 命令。

我的宏还支持一个额外的功能：自动生成文档。

如果参数 `:doc` 的值是 auto 的话（不需要加单引号），宏将根据按键和调用的函数名自动 生成文档，然后设置 `:use-minibuffer t` 的话，就可以把按键的提示放进 mini-buffer 了。

另外，这个宏还有一个额外的选项 `:with-full-arguments` ，默认为 nil，如果设置为 t， 则宏生成的函数将与原函数有相同的形参列表和 interactive 形式。同时，如果有一些使用 `evil-define-command` 定义的没法获取 interactive 命令的函数的话，会自动加上一个 interactive 命令。使用这个选项的宏在展开的时候必须确保原函数的定义已经完全加载， 否则会报错。

# 使用<a id="sec-"></a>

## 另一个宏<a id="sec-"></a>

针对一些键绑定函数如 `define-key` 没法一次定义多个键绑定的问题，我定义了一个宏。

```emacs-lisp
(defmacro liu233w|bind-keys (binding-list &rest func-and-args)
  "从列表自动生成多个键绑定命令
语法为：
\(liu233w|bind-keys ((\"mn\" 'func1) (\"mp\" 'func2))
                       define-key evil-visual-state-map)
键绑定会自动添加，不会自动调用 kbd。这个宏会生成多个键绑定函数的调用，
每次都使用 binding-list 中的一项（去掉括号）放在函数调用的最后。
除了 binding-list 以外，请使用和直接调用键绑定函数时相同的语法"
  `(progn
     ,@(mapcar #'(lambda (item)
                   (append func-and-args item))
               binding-list)))
```

这样绑定按键可以方便一些，并且支持多种键绑定函数。

## 示例 1<a id="sec-"></a>

绑定翻页键：

```emacs-lisp
;; 一个 micro-state，用来快速翻页
(mms|define-multiple-micro-state
 liu233w/view
 :use-minibuffer t
 :doc "`d' scroll-down `u' scroll-up `f' scroll-page-down `b' scroll-page-up"
 :bindings
 ("d" evil-scroll-down)
 ("u" evil-scroll-up)
 ("f" evil-scroll-page-down)
 ("b" evil-scroll-page-up))

(liu233w|bind-keys
 (((kbd "C-f") 'liu233w/view:evil-scroll-page-down-then-enter-micro-state)
  ((kbd "C-b") 'liu233w/view:evil-scroll-page-up-then-enter-micro-state)
  ((kbd "C-u") 'liu233w/view:evil-scroll-up-then-enter-micro-state)
  ((kbd "C-d") 'liu233w/view:evil-scroll-down-then-enter-micro-state))
 evil-global-set-key 'normal)
```

翻页之后会自动进入 micro-state。并且这种定义对于原来的函数（ `evil-scroll-page-up` ） 是没有影响的。

## 示例 2<a id="sec-"></a>

定义 evil-mc 的按键：

```emacs-lisp
(mms|define-multiple-micro-state
 liu233w/emc
 :doc auto
 :use-minibuffer t
 :with-full-arguments t
 :bindings
 ("n" evil-mc-make-and-goto-next-match)
 ("j" evil-mc-skip-and-goto-next-match)
 ("p" evil-mc-make-and-goto-prev-match)
 ("k" evil-mc-skip-and-goto-prev-match))

(liu233w|bind-keys
 (((kbd "mn") #'liu233w/emc:evil-mc-make-and-goto-next-match-then-enter-micro-state)
  ((kbd "mp") #'liu233w/emc:evil-mc-make-and-goto-prev-match-then-enter-micro-state))
 define-key evil-visual-state-map)
```

生成的文档是这样的：

    `n' evil-mc-make-and-goto-next-match  `j' evil-mc-skip-and-goto-next-match  `p' evil-mc-make-and-goto-prev-match  `k' evil-mc-skip-and-goto-prev-match

# 经验<a id="sec-"></a>

我在写这些代码的时候发现了一些好东西，在这里分享一下：

1.  `defalias` 可以生成函数的别名，对这个别名使用 `advice-add` 的话只会把新函数 绑定到别名上，原来的函数不受影响。
2.  `interactive-form` 可以获得某个函数的 interactive 形式。
3.  `intern` 可以从一个字符串生成一个 symbol。
4.  `help-function-arglist` 可以获得某个函数的的参数列表。
5.  `command-execute` 可以像执行命令一样执行一个 interactive 函数，调用的函数不需要 保留参数信息， `command-execute` 会将要调用的函数需要的参数自行传递给它。
