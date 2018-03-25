---
date: 2016/08/20 00:00:00
updated: 2018/03/25 22:26:29
tags:
- acm
categories:
- acm
title: OJ 中使用 fread 加速读入
---

关于 fread 加速读入的说明请参考 [我之前的博文](http://liu233w.github.io/acm/2016/07/25/hdu-4825/#orgheadline8)。

在使用这段代码的时候我发现在某些题目上容易出错导致 TLE，比如 POJ-1330。

有一些 OJ 的用例可能在两个数字之间隔了多个分隔符，导致输入出错。所以我重新写了一个模板， 贴在这里。现在 sread 函数可以正确处理两个整型之间相隔了多个分隔符的情况，但是仍然不能 处理小数。

```c++
const int MAXBUF = 1000000 << 2;
char buf[MAXBUF];
char* startpoint = buf;
int BUFLEN;

template<typename T>
inline void sread(T &res)
{
    res = 0;

    while (startpoint < buf + BUFLEN && !isdigit(*startpoint))
        ++startpoint;

    for (;*startpoint&&startpoint < buf + BUFLEN;++startpoint)
    {
        if (!isdigit(*startpoint))
        {
            ++startpoint;
            return;
        }
        else
            res = res * 10 + *startpoint - '0';
    }
}

inline void read_init()
{
    BUFLEN = fread(buf, sizeof(char), MAXBUF, stdin);
    if (BUFLEN >= MAXBUF)
        throw int();
}
```

在 main 函数的开头调用 `read_init()` 初始化，之后使用 sread 读取数据即可。
