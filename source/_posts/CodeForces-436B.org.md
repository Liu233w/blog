---
date: 2016/07/10 00:00:00
updated: 2018/03/25 22:26:29
tags:
- acm
categories:
- acm
title: CodeForces 436B Om Nom and Spiders
---

- [题目：](#sec-)
- [代码：](#sec-)
- [解析&吐槽：](#sec-)


# 题目：<a id="sec-"></a>

<p class="verse">
B. Om Nom and Spiders<br />
time limit per test<br />
3 seconds<br />
memory limit per test<br />
256 megabytes<br />
input<br />
standard input<br />
output<br />
standard output<br />
<br />
Om Nom really likes candies and doesn't like spiders as they frequently steal candies. One day Om Nom fancied a walk in a park. Unfortunately, the park has some spiders and Om Nom doesn't want to see them at all.<br />
<br />
The park can be represented as a rectangular n × m field. The park has k spiders, each spider at time 0 is at some cell of the field. The spiders move all the time, and each spider always moves in one of the four directions (left, right, down, up). In a unit of time, a spider crawls from his cell to the side-adjacent cell in the corresponding direction. If there is no cell in the given direction, then the spider leaves the park. The spiders do not interfere with each other as they move. Specifically, one cell can have multiple spiders at the same time.<br />
<br />
Om Nom isn't yet sure where to start his walk from but he definitely wants:<br />
<br />
&#xa0;&#xa0;&#xa0;&#xa0;to start walking at time 0 at an upper row cell of the field (it is guaranteed that the cells in this row do not contain any spiders);<br />
&#xa0;&#xa0;&#xa0;&#xa0;to walk by moving down the field towards the lowest row (the walk ends when Om Nom leaves the boundaries of the park).<br />
<br />
We know that Om Nom moves by jumping. One jump takes one time unit and transports the little monster from his cell to either a side-adjacent cell on the lower row or outside the park boundaries.<br />
<br />
Each time Om Nom lands in a cell he sees all the spiders that have come to that cell at this moment of time. Om Nom wants to choose the optimal cell to start the walk from. That's why he wonders: for each possible starting cell, how many spiders will he see during the walk if he starts from this cell? Help him and calculate the required value for each possible starting cell.<br />
Input<br />
<br />
The first line contains three integers n, m, k (2 ≤ n, m ≤ 2000; 0 ≤ k ≤ m(n - 1)).<br />
<br />
Each of the next n lines contains m characters — the description of the park. The characters in the i-th line describe the i-th row of the park field. If the character in the line equals ".", that means that the corresponding cell of the field is empty; otherwise, the character in the line will equal one of the four characters: "L" (meaning that this cell has a spider at time 0, moving left), "R" (a spider moving right), "U" (a spider moving up), "D" (a spider moving down).<br />
<br />
It is guaranteed that the first row doesn't contain any spiders. It is guaranteed that the description of the field contains no extra characters. It is guaranteed that at time 0 the field contains exactly k spiders.<br />
Output<br />
<br />
Print m integers: the j-th integer must show the number of spiders Om Nom will see if he starts his walk from the j-th cell of the first row. The cells in any row of the field are numbered from left to right.<br />
Examples<br />
Input<br />
<br />
3 3 4<br />
...<br />
R.L<br />
R.U<br />
<br />
Output<br />
<br />
0 2 2<br />
<br />
Input<br />
<br />
2 2 2<br />
..<br />
RL<br />
<br />
Output<br />
<br />
1 1<br />
<br />
Input<br />
<br />
2 2 2<br />
..<br />
LR<br />
<br />
Output<br />
<br />
0 0<br />
<br />
Input<br />
<br />
3 4 8<br />
....<br />
RRLL<br />
UUUU<br />
<br />
Output<br />
<br />
1 3 3 1<br />
<br />
Input<br />
<br />
2 2 2<br />
..<br />
UU<br />
<br />
Output<br />
<br />
0 0<br />
<br />
Note<br />
<br />
Consider the first sample. The notes below show how the spider arrangement changes on the field over time:<br />
<br />
<br />
...        ...        ..U       ...<br />
R.L   ->   .\*U   ->   L.R   ->  ...<br />
R.U        .R.        ..R       ...<br />
<br />
Character "\*" represents a cell that contains two spiders at the same time.<br />
<br />
&#xa0;&#xa0;&#xa0;&#xa0;If Om Nom starts from the first cell of the first row, he won't see any spiders.<br />
&#xa0;&#xa0;&#xa0;&#xa0;If he starts from the second cell, he will see two spiders at time 1.<br />
&#xa0;&#xa0;&#xa0;&#xa0;If he starts from the third cell, he will see two spiders: one at time 1, the other one at time 2.<br />
</p>

# 代码：<a id="sec-"></a>

```c++
// oj.cpp : 定义控制台应用程序的入口点。
//
//#include <iostream>
#include <cstdio>
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

int m, n;

int k;

int res[2005];

char str[2005];

int main()
{
    scanf("%d %d %d\n", &n, &m, &k);

    for (int r = 0;r < n;++r)
    {
        scanf("%s", str);
        for (int c = 0;c < m;++c)
        {
            char chr = str[c];
            switch (chr)
            {
            case '.':
                break;
            case 'L':
                if (c - r >= 0)
                    ++res[c - r];
                break;
            case 'R':
                if (c + r < m)
                    ++res[c + r];
                break;
            case 'D':
                break;
            case 'U':
                if (r % 2==0)
                    ++res[c];
                break;
            }
        }
    }

    printf("%d", res[0]);
    for (int i = 1;i < m;++i)
        printf(" %d", res[i]);
    printf("\n");

    return 0;
}
```

# 解析&吐槽：<a id="sec-"></a>

题意如下：一个方形区域中有许多蜘蛛（除了第 0 行之外），每个蜘蛛都会朝某个固定的方向移动。 怪物从最上面一行的某个格子开始往下走，单位时间内向下走一格（只能向下，不能向别的方向）， 同时每个蜘蛛也都向固定的方向移动一格，直到移动出方形区域。一个格子里可以有任意数量的 蜘蛛。问如果怪物从第一行的第一格、 第二格……第 m 格开始向下移动，总共会遇到多少蜘蛛。每个用例打印出 m 个数，分别表示从 某一格开始向下移动时遇到的蜘蛛总数。只有怪物和蜘蛛落到同一个格子里的时候才算是遇到， 在移动的时候经过对方不算是碰面。

这道题用模拟的方法肯定是会超时的，但是注意到蜘蛛只能朝一个方向移动，哪个格子会遇到蜘蛛 是可以用数学方法计算出来的，我们可以只用一个数组记录某一列遇到的蜘蛛总数， 扫描一遍整个区域，检测蜘蛛的开始位置和移动方向，然后遇到蜘蛛的次数可以通过如下规律计算出来：

-   **向下走的蜘蛛:** 这种情况肯定是不会和怪物碰面的，因为蜘蛛和怪物向下移动的速度一样快。
-   **向上走的蜘蛛:** 这时候需要蜘蛛的所在行数来判断。怪物从第 0 行开始向下移动，蜘蛛如果 在偶数行，它和怪物之间就相隔了奇数行，一定会在移动的过程中和怪物碰面，因此把这一列 遇到蜘蛛的总数加一；如果在奇数行，蜘蛛和怪物不会在格子中碰面。
-   **向右走的蜘蛛:** 假设蜘蛛目前在第 r 行，第 c 列。如果怪物可以和这个蜘蛛碰面，它一定要 向下移动 r 此，同时这个蜘蛛会向右移动 r 次，在第 c+r 列碰面。如果 c+r 没有超过最 大列数的话，就把此列遇到的蜘蛛总数加一。
-   **向左走的蜘蛛:** 和上面的情况对称，在第 c-r 列相遇。
