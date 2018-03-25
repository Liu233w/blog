---
date: 2017/02/24 00:00:00
updated: 2018/03/25 22:48:19
tags:
- sicp
categories:
- blog
title: SICP 1.16 题 题解
---

- [题目描述](#sec-)
- [分析](#sec-)
- [源代码](#sec-)

刚刚做出来 sicp 里的 1.16 题，深有感触。看了一下[网上的题解](http://sicp.readthedocs.io/en/latest/chp1/16.html) ，并没有题目的分析过 程，因此把我的分析过程在这里记录了一下。

# 题目描述<a id="sec-"></a>

根据书中给出的关系 ![img](sicp-1-16/1.png) ，并且使用一个不变量记录中间结果，写出对数步数内迭代计算幂的函数。

# 分析<a id="sec-"></a>

题目中说到令 ![img](sicp-1-16/2.png) ，而 a 的初始值为 1，在最后 a 为运算的 结果（b 的 n 次方）。可得公式：![img](sicp-1-16/3.png) 。因此推知迭代终点为 `n = 0` 。

令初始公式为 ![img](sicp-1-16/4.png) 。

-   当 `n = 0` 时，迭代结束
-   当 n 为偶数时，可得 ![img](sicp-1-16/5.png) ，在结果中可以把 b<sup>2</sup> 看作新 的 b，把 n/2 看作新的 n，然后继续迭代
-   当 n 为奇数时，n-1 为偶数或者 0，有 ![img](sicp-1-16/6.png) ，可以把 a\*b 看作新的 a，把 b<sup>n-1</sup> 看作新的 b。

由此，所有的情况已经分析完毕。

# 源代码<a id="sec-"></a>

```racket
#lang racket
(define (expt-iter b n a)
  [cond ([= n 0] a)
        ([even? n]
         (expt-iter (* b b) (/ n 2) a))
        (else
         (expt-iter b (- n 1) (* a b)))])

(define (expt b n)
  (expt-iter b n 1))
```
