---
date: 2016/07/10 00:00:00
updated: 2018/03/25 22:26:29
tags:
- acm
categories:
- acm
title: Codeforces-393A Nineteen
---

- [题目：](#sec-)
- [代码：](#sec-)
- [解析&吐槽：](#sec-)


# 题目：<a id="sec-"></a>

<p class="verse">
<br />
A. Nineteen<br />
time limit per test<br />
1 second<br />
memory limit per test<br />
256 megabytes<br />
input<br />
standard input<br />
output<br />
standard output<br />
<br />
Alice likes word "nineteen" very much. She has a string s and wants the string to contain as many such words as possible. For that reason she can rearrange the letters of the string.<br />
<br />
For example, if she has string "xiineteenppnnnewtnee", she can get string "xnineteenppnineteenw", containing (the occurrences marked) two such words. More formally, word "nineteen" occurs in the string the number of times you can read it starting from some letter of the string. Of course, you shouldn't skip letters.<br />
<br />
Help her to find the maximum number of "nineteen"s that she can get in her string.<br />
Input<br />
<br />
The first line contains a non-empty string s, consisting only of lowercase English letters. The length of string s doesn't exceed 100.<br />
Output<br />
<br />
Print a single integer — the maximum number of "nineteen"s that she can get in her string.<br />
Examples<br />
Input<br />
<br />
nniinneetteeeenn<br />
<br />
Output<br />
<br />
2<br />
<br />
Input<br />
<br />
nneteenabcnneteenabcnneteenabcnneteenabcnneteenabcii<br />
<br />
Output<br />
<br />
2<br />
<br />
Input<br />
<br />
nineteenineteen<br />
<br />
Output<br />
<br />
2<br />
<br />
</p>

# 代码：<a id="sec-"></a>

```c++
// oj.cpp : 定义控制台应用程序的入口点。
//

#include <iostream>
#include <iomanip>
#include <algorithm>
#include <sstream>
#include <string>
#include <numeric>
#include <cstring>
#include <map>
#include <set>
#include <utility>
#include <map>
#include <list>
#include <queue>
#include <functional>
#include <bitset>
#include <stack>
#include <deque>

using namespace std;

int main()
{
    string L;
    cin >> L;
    int n = 0, i = 0, e = 0, t = 0;
    for (char c : L)
    {
        switch (c)
        {
        case 'n':
            ++n;
            break;
        case 'i':
            ++i;
            break;
        case 'e':
            ++e;
            break;
        case 't':
            ++t;
            break;
        default:
            break;
        }
    }

    int minn = min({ i, e / 3, t, (n - 1) / 2 });

    cout << minn << endl;
}
```

# 解析&吐槽：<a id="sec-"></a>

一道水题。计算一段字符串中可以含有几个 nineteen 这个单词。需要注意的是这个单词的头和尾都是 n， 是可以连起来的。也就是说 `nineteenineteen` 算作两个单词。算法比较简单，统计所有的 n i e t 的个数， 除以它们在 nineteen 中出现的频率即可。对于 `n` ，我们可以列一个方程，两个单词需要 5 个 n， n 个单词需要 `n*3-(n-1) = 2n+1` 个 n，反之 x 个 n 最多可以拼成 `(x-1)/2` 个单词。找出所有的 字母可以拼出的单词数的最小值即可。
