---
date: 2016/07/15 00:00:00
updated: 2018/03/25 22:48:19
tags:
- clojure
- c++
- lambda
categories:
- blog
title: 用 C++11 的 lambda 实现简易闭包
---

- [原先的博文](#sec-)
- [当天更新](#sec-)


# 原先的博文<a id="sec-"></a>

今天突然想到可以用 lambda 实现一个类似函数式编程当中的累加器的东西，我首先想到的是如下的代码：

```c++
auto func = [](int beg)
{
    return [&]() {return ++beg;};
};

auto f1=func(0);

cout << f1() <<" "<<f1()<<endl;
```

但是这段代码输出的是乱码！因为 c++的 lambda 没有闭包功能，我们返回的函数只能保存捕获的值或 引用。这里我们传递给 func 函数的 beg 参数在它返回 lambda 之后就被销毁了，而它返回的函数会在 beg 被销毁之后调用。因此我们访问的 beg 就是未定义的内存了。

那怎么解决这个问题呢？用指针保存！

```c++
auto func = [](int beg)
{
    int *a = new int(beg);
    return [=]() {return ++*a;};
};

auto f1 = func(0);

cout << f1() << " " << f1() << endl;
```

我们在 func 函数里面定义了一个指针，然后把指针按值传递给返回的 lambda（注意那个 `&` 被改成了 `=` ）。这样保存的值就不会在 func 函数超出作用域之后被销毁了，可以在外部正常访问。 同时我们可以生成任意数量的累加器，比如 f2 f3 等等，每个累加器的初始值都可以是不同的， 而且相互独立。

但是这种写法仍然有两个问题：

1.  指针 a 没有办法被销毁，在函数结束后变成了野指针。这个问题我会在下文解决。
2.  上个程序的输出是“2 1”，就好像先运行了 `*a==2` 的情况，后运行了 `*a==1` 的情况一样。 我不知道为什么会这样，但是如果把两个函数调用放到两个 cout 的语句中，输出就是“1 2”， 正常了。我把这个问题放到了[ 知乎](https://www.zhihu.com/question/48510152) 上，希望有人可以给我解答。

现在我们来解决野指针的问题，这是我的代码：

```c++
auto func = [](int beg)
{
    int *a = new int(beg);
    return make_pair([=]() ->int {return ++*a;}, shared_ptr<int>(a));
};

auto f1 = func(0);

cout << f1.first() << " " << f1.first() << endl;
```

这里我们把指针放到了一个 shared\_ptr 里面，然后把这个智能指针和函数打包到了一个 pair 里面。 func 函数返回了 pair，如果我们想要调用累加器函数，就必须要写成 `f1.first()` 了。现在， 指针的生命周期和 lambda 的生命周期绑定到了一起，如果我们复制了 lambda 函数，智能指针也会 一起被复制。只有在所有的 pair 被销毁之后（这时候也没法访问 lambda 函数了），指针才会被 销毁。

我们可以把上一个代码包装一下，做成一个可以接受任意数量的参数的闭包。

```c++
#define comma ,
#define clojure(arg, v, init_v, f) [] arg \
{\
    struct str \
    v *s = new str init_v ; \
    return std::make_pair([=]()f, std::shared_ptr<str>(s)); \
}

auto func = clojure((int beg, int end), { int a; int b; },
    {beg comma end}, { return (++s->a) + s->b; });

auto f1 = func(0,1);

cout << f1.first() << " "
     << f1.first() << endl;
```

clojure 是一个宏（我承认我被 lisp 给玩坏了\_(:зゝ∠)\_）。我们把多个参数放到了一个 struct 里面，就可以保存多个变量了。 宏的参数：

-   **第一个参数，arg:** 生成的“累加器的生成器”的形参（当然不一定非得是累加器），不管有几个参数都 必须用一个括号包起来（没有参数就要一个空括号）。必须要写参数类型。
-   **第二个参数，v:** struct 的 body，要用大括号包起来。要写参数类型，参数之间用分号隔开。
-   **第三个参数，init\_v:** 给 struct 赋值的时候依次赋给 body 里面各个变量的值，需要用大括号 包起来，多个参数时要在参数之间用关键字 comma 隔开。我这里用的是 c++11 里面 struct 的大括号赋值法，如果在 struct 里面加上函数把它变成类的话，貌似就没法这么赋值了。
-   **第四个参数，f:** 返回的 lambda 的函数体，如果需要声明此 lambda 的返回值的话，可以把箭头的 声明放在大括号前面。想要使用 struct 的变量，可以用 `s->v 中的变量名` 的方法来访问。

接下来我们试一下指针是否会被销毁。

```c++
#include "stdafx.h"

#include <iostream>
#include <memory>
#include <utility>

using namespace std;

#define comma ,
#define clojure(arg, v, init_v, f) [] arg \
{\
    struct str \
    v *s = new str init_v ; \
    return std::make_pair([=]()f, std::shared_ptr<str>(s)); \
}

struct test
{
    ~test()
    {
        cout << "I'm dead." << endl;
    }
};

int main()
{
    auto func = clojure((int beg), { int a; test t; },
        {beg comma test()}, { return ++(s->a); });

    auto f1 = func(0);

    cout << f1.first() << " "
         << f1.first() << endl;


    return 0;
}
```

这段程序是在 visual studio 2015 上编译运行的，结果是：

<p class="verse">
I'm dead.<br />
2 1<br />
I'm dead.<br />
请按任意键继续. . .<br />
</p>

可以看到确实是删掉了。啊嘞Σ(⊙▽⊙"a 怎么销毁了两次？这下我就不明白了。以后再研究吧\_(:зゝ∠)\_

# 当天更新<a id="sec-"></a>

输出 2 1 的原因我明白了，是求值顺序未定义，我居然忘了这么基础的知识点\_(:зゝ∠)\_。 对于如下的代码，一个表达式中各个值的求值顺序是未定义的：

```c++
int i = 0;
cout << ++i << " " << ++i << endl;
```

我的编译器上输出是 `2 2` 。因为标准没有规定对于这两个 i 是从左到右求值、从右到左求值还是 先全求值完再直接显示结果。

至于显示销毁了两次的原因，第一次是因为我们在 func 函数中用 test 的默认构造函数生成了一个 test 并将它复制给了指针中的结构体里的 test。这个语句结束之后第一个生成的 test 就被销毁了。

```c++
auto func = clojure((int beg), { int a; test t; },
    {beg}, { return ++(s->a); });
```

上面的是修改版的 func，删去了第二行的 `comma test()` ，就略去了复制赋值的环节，只会在 new 对象的时候生成 test。现在 test 就只会销毁一次了。

看来我对 c++掌握的还不够啊\_(:зゝ∠)\_
