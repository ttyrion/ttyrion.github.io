---
layout:         page
title:         【Algorithm】理解KMP算法
subtitle:       
date:           2018-11-24
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 最近看了两篇博客，把KMP算法讲的比较清楚，但对于喜欢问“为什么”的我，其中间有一些直接给的结论我忍不住问“为什么是这样？”，当然，这个问题还得自己来回答。这篇博客就主要是记录我自己的理解以及回答自己几个“为什么”。最后会贴上上述两篇博客的链接，都是神人，能把KMP讲的比算法书还清晰。

### Naive-String-Match
朴素字符串匹配最简单最直接的匹配算法：既然要在一个字符串S中找到匹配模式串P（假设前提S.length >= P.length）的位置，那就把S中的每个可能的位置（只有前面 S.length-P.length+1 个位置才有可能是我们要的）都进行匹配测试。
```C
int naive_find_first_of(const char *str, const char *pattern) {
    const int sn = strlen(str);
    const int pn = strlen(pattern);
    int si = 0;
    
    /* Only the first (S.length-P.length+1) position maybe the solution. */
    for(; si <= sn - pn; ++si) {
        int pi = 0;
        for(; pi < pn; ++pi) {
            if(str[si+pi] != pattern[pi]) {
                break;
            }
        }

        /* match */
        if (pi == pn) {
            return si;
        }
    }

    return -1;
}
```

### Naive-String-Match 为什么会低效？
上面的Naive-String-Match算法的问题在于太“Naive”，埋头苦干（匹配），却没有利用每次匹配得到的信息。什么信息？下面就来说说。

比如下面的一个字符串匹配过程：
![naivesort1](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/algorithm/kmp/naivesort1.png) 
匹配si=6，pi=4时，发现 S[si+pi] != P[pi]，按照朴素匹配算法，则开始下一轮匹配，即si=7,pi=0,如下图：
![naivesort2](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/algorithm/kmp/naivesort2.png) 

实际上，新的S[si+pi]和P[pi]肯定不能匹配，因为在之前的匹配中，已经知道S[si+pi]=S[7]='D'，且P[pi]=P[0]='C'。**Naive-String-Match** 没有利用这些已获知的信息。

### Knuth–Morris–Pratt algorithm
应用KMP算法时，上面第一幅图中匹配失败时，会进行如下匹配：
![naivesort3](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/algorithm/kmp/naivesort3.png) 

仔细对比，在KMP算法中，并非简单地把si自增1，pi设置为0，开始新一轮匹配。KMP算法根据已获知的信息，直接开始si=8，pi=2的匹配过程。**这个过程怎么发生的？** 就是我这里要说的重点。

**注意：** 很多讲KMP算法的资料都说KMP在匹配失败时，不会改变字符串S的遍历过程，仅仅只是**移动** 模式串。这个说法是对的！我上面的匹配图中，好像si被改了，si从6变为8！实际上，对字符串S的遍历依然是不变的。因为我们访问的字符串S的位置为si+pi = 8+2 = 10。在前面匹配失败时，si+pi = 6 + 4 = 10。很明显，我的说法和他们是一样的，字符串S的索引完全没变，我们接下来还是会匹配S[10]（和P[pi]）。

那么为什么我要画这样的图，看起来好像跟其他地方讲的不一样? 其实我只是为了突出 **我们要求的解就是某个si**。我的代码也会着重于这一点，我认为算法都是一样的，代码可以有些变化。

现在匹配字符串的问题已经转化为：“在匹配失败的时候，模式串应该向右移动多少？” 这个问题其实就是想问“匹配失败的时候，应该如何改变pi的值，用新的P[pi]再去和之前的S[10]去匹配？” 这个问题用一个式子描述就是：求匹配P[pi]失败时的next(pi)，可以拿P[next(pi)]去跟S[10]匹配。

我们再次观察上面那个匹配，为什么KMP算法匹配中可以把模式串向右移动两位？因为它知道移动2位后，S[8]和P[0]匹配，而S[9]和P[1]匹配，现在要做的是匹配P[2]和S[10]，也就是说在这里，next(4)=2。算法如何知道？因为从已经匹配过的信息中，它知道S[8]=P[2],S[9]=P[3]，而模式串中P[0]=P[2],P[1]=P[3]。因此，KMP算法可以直接移动模式串（2位），使得S[8]、S[9]分别与P[0]、P[1]对齐，从S[10]和P[2]开始匹配。意思就是，当某个S[n] 和P[4] 匹配失败时候，就再把S[n]和P[next(4)],即P[2]进行匹配。从图示看来，这就相当于把模式串向右移动了4-next(4)个位置。对于这一过程，我认为**最好是理解成**：在匹配S[n]和P[pi]失败时，如何求得next(pi)，使得我们可以直接拿S[n]和P[next(pi)]来匹配。注意这里说的是“直接拿S[n]”，这表示我们没有回溯字符串S的索引，即便si因为pi的变化被间接增大了。

但是还有一种情况！

如果S[n]和P[1]匹配失败了，那么可以拿P[0]再来和S[n]进行匹配。可是如果S[n]和P[0]匹配失败，又怎么办？这里明显不能再保持n不变，找一个P[next(0)]来进行匹配。此时只能把n加1，即要匹配S[n+1]和P[0]。为了依然能用next来描述这个过程，我们定**next(0)=-1**;定next(0)=-1是有用处的：这涉及从另一个角度，即“移动模式串P”来理解KMP匹配过程：

前面说最好把KMP匹配的过程理解成：在匹配S[n]和P[pi]失败时，如何求得next(pi)，使得我们可以直接拿S[n]和P[next(pi)]来匹配。我说“最好”，有两点意思，一是为了突出next的意义，它就是为了确定下一个拿来匹配的模式串P的索引。二是这个过程当然可以有另一种理解方式。但是这个“最好的理解”在前面说的定next(0)=-1的情况下，又似乎不对：我们不能拿S[n]和一个没有意义的P[-1]来匹配。此时“另一种理解方式”就有用了：我们可以把匹配S[n]和P[next(pi)]的过程看成是，当匹配S[n]和P[pi]失败时，把模式串向右**移动** pi-next(pi)个位置。当匹配S[n]和P[0]失败时，就把模式串P向右移动0-next(0)个位置，也就是移动1一个位置，这与实际匹配过程是符合的。

上面是理解KMP算法匹配过程。那么现在问题就剩下：next(pi)是多少怎么求解？

### next数组
仔细分析上述KMP匹配的过程，可以发现在匹配P[pi]失败时的next(pi)，是模式串的一个属性，与字符串S无关！因为这个next(pi)正是模式串中的子串P[0,pi-1] （表示模式串P的0至pi-1子元素）中相同的前缀和后缀的最大字符个数，这里称之为子串的“公共前缀后缀的最大字符个数”。换种说法就是，子串P[0,pi-1] 的头N个字符和尾N个字符刚好完全一致，并且头N+1个字符和尾N+1个字符不一致，那么next(pi)就等于N。举例来说，有个子串“ABACABA”，它有个前缀A和后缀A，还有前缀ABA和后缀ABA，那么它的“公共前缀后缀的最大字符个数”就是3。

至此，对于一个模式串，我们知道有一组next值，那就把它定义成一个数组，即**next数组**。并且我们知道数组的元素个数就是模式串的字符个数。我们还知道next[0]= -1。从上面的讲解中，我们还知道，next数组只跟模式串P有关。

现在我们该考虑的问题是怎么求解一个模式串剩下的next值。既然我们已经知道next[0]= -1，那么试试**数学归纳法**：

假设next[pi] = k, next[pi+1]的值是多少呢？
因为next[pi] = k， 就表示模式子串 P[0,k-1] 和 P[pi-k,pi-1] 是一致的。那么求next[pi+1]就有两种情形：

情形一： 如果 P[k] = P[pi]，就知道模式子串 P[0,k] 和 P[pi-k,pi] 是一致的，这是模式子串 P[0,pi] 的“公共前缀后缀的最大字符个数”，也就是next[pi+1]。所以next[pi+1] = k+1。

情形二： 如果 P[k] != P[pi], next[pi+1]一定是子串 P[0,k]的“公共前缀后缀的最大字符个数”。但是我这里不打算给这么一个结论就算了，我要证明一下这个结论。令 next[pi+1] = k2，并且假设 k2 > k，即 k2 >= k+1。那么我们知道模式子串 P[0,k2-1] 和 P[[pi+1-k2,pi] 是一致的，分别去掉他们最后一个字符，得到的子串 P[0,k2-2] 和 P[[pi+1-k2,pi-1] 也是一致的。这说明模式子串P[0,pi-1]还有一个公共的前缀后缀长度是 P[0,k2-2] 或 P[[pi+1-k2,pi-1] 的长度,即 k2-1。而next数组的定义决定了 next[pi] 应该满足： next[pi] = max(k,k2-1)。因为 k2 >= k+1，所以 k2-1 >= k,可以推论出 max(k,k2-1) = k2-1。进一步可以的出结论 next[pi] = k2-1。而已知 next[pi] = k，所以得出 k2-1=k,即 k2 = k+1。也就是说我们推论出 next[pi+1] = k+1。这表示子串 P[0,k] = P[pi-k,pi]，也就是说 P[k]=P[pi]。但是这与我们的前提假设 P[k] != P[pi] 相悖。因此前面“假设 k2 > k”是不成立的。**反证法结束。**
 
 **所以结论是：如果 P[k] != P[pi], k2 一定小于或等于 k，即 next[pi+1]一定小于或等于 next[pi]。**

上面的结论可以告诉我们，如果令 next[pi+1]=k2，那么k2 <= k。这表示模式子串 P[0,k2-1] 和 P[pi-k2+1,pi] 是一致的，分别拆开他们最后一个字符，得到的子串 P[0,k2-2] 和 P[pi-k2+1,pi-1] 是一致的（等式一），并且P[k2-1]=P[pi](等式二)。而已知 next[pi] = k，即模式子串 P[0,k-1] 和 P[pi-k,pi-1] 是一致的。因为 k2 <= k, P[0,k2-2] 一定是 P[0,k-1] 的子串，同样， P[pi-k2+1,pi-1] 一定是 P[pi-k,pi-1] 的子串。而从 P[0,k-1] = P[pi-k,pi-1] 可以推论出 P[k-k2+1,k-1] = P[pi-k2+1,pi-1]（等式三）。**这里这些子串相等的推论看起来很晦涩，其实就是数子串长度。**至此，我们有以下几个等式：
```C
P[0,k2-2] = P[pi-k2+1,pi-1]      // 等式一
P[k2-1]=P[pi]                    // 等式二
P[k-k2+1,k-1] = P[pi-k2+1,pi-1]  // 等式三

```
上面一、三这两个等式可以推论出 P[0,k2-2] = P[k-k2+1,k-1]。 而 P[0,k2-2] 或 P[k-k2+1,k-1] 正是模式子串 P[0,k-1] 的最长的公共前缀和后缀，也就是说 next[k] = k2-1，因此 k2 = next[k] + 1。再结合第二个等式，可以推论出P[next[k]] = P[pi]。这就告诉我们，我们要求k2，其实就是求出 next[k]，**并且要满足 P[next[k]] = P[pi]** 。如果不满足，就要继续求 next[next[k]], 并且要满足 P[next[next[k]]] = P[pi] 。如此递归,直到满足条件或者某次 next[kn] = -1，则不继续递归：next[pi+1] = next[kn] + 1 = 0 。

**所以得出结论， 令 next[pi] = k：**
1. 如果 P[k] == P[pi]， 则 next[pi+1] = k + 1 = next[pi] + 1 。 
2. 如果 P[k] != P[pi]， 则 当 P[next[k]] = P[pi] 时， next[pi+1] = next[k] + 1；否则，继续求出next[next[k]], 若它满足P[next[next[k]]] = P[pi]，则 next[pi+1] = P[next[next[k]]] + 1；如此递归，直至求出满足P[kn] = P[pi]，或者 kn = -1 时为止。**请注意，这里只是递归 next[k], "+1" 是不递归的。**

上面的结论用代码表示就是：
```C
int* kmp_get_next(const char *p) {
    const int pn = strlen(p);
    if(pn <= 0) {
        return nullptr;
    }

    int *next = new int[pn];
    next[0] = -1;
    memset(next+1, 0, sizeof(int)*(pn-1));

    for(int pi = 1; pi < pn; ++pi) {
        int k = next[pi-1];
        while(true) {
            if (k == -1 || p[k] == p[pi-1]) {
                next[pi] = k + 1;
                break;
            }
            else {
                k = next[k];
            }
        }
    } 

    return next;
}

```

至此，我们已经求出了一个模式串的next数组了。

### KMP字符串匹配
KMP算法匹配和朴素匹配算法的区别，在于匹配失败时的处理。KMP不是简单粗暴地把 si 向后移动一个位置，同时设置 pi 为0，而是把 pi 设置为next[pi]，这在效果上，相当与把 si 设置为 si+pi-next[pi]。本质上，KMP相当于是在一次匹配失败时，设置 pi 为0，但是把si向后移动 pi-next[pi] 个位置继续匹配。当 pi-next[pi] > 1 时，KMP算法就比朴素匹配算法效率高了。

用KMP算法匹配模式串的代码如下，上面说过，我在代码里面还是会着重展现si的操作：

```C

int kmp_find_first_of(const char *str, const char *pattern) {
    const int sn = strlen(str);
    const int pn = strlen(pattern);
    int *next = kmp_get_next(pattern);
    int si = 0;
    for(; si <= sn - pn;) {
        int pi = 0;
        for(; pi < pn;) {
            if(str[si+pi] != pattern[pi]) {
                /*Changing pi actually moves si.*/
                si += pi - next[pi];
                pi = next[pi];
                break;
            }
            ++pi;
        }

        /* match */
        if (pi == pn) {
            delete next;
            return si;
        }
    }

    delete next;
    return -1;
}

```

