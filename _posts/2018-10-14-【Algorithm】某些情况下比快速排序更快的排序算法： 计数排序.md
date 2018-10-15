---
layout:         page
title:         【Algorithm】某些情况下比快速排序更快的排序算法： 计数排序
subtitle:       一种简单场景下比快排更高效的排序算法
date:           2018-10-14
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---


偶然看到一个微信群里分享的一篇技术文章，标题叫“计数排序”，当时就懵了：没听说过还有一种叫做计数排序的算法（孤陋寡闻的我）。看完此篇文章后特别感慨：算法原理很简单，但是在一些场景下，就会比快速排序更快。下面就开始说明“计数排序”。

问题如下：有一个整数序列，包含10个随机整数，取值范围在0到15之间，要求对这个整数序列排序。

这里就不说快速排序了，直接上计数排序。

**计数排序的原理**如下：建立一个用于排序的辅助数组B，让待排序数组A的元素对应到数组B的索引（0-N），并对它计数，计数结果填充在数组B中。假设数组B的某个元素E的索引是index，元素值是value。就表示待排序数组A中有value个index（index属于0-N范围）。

假设待排序数组是这样：2,5,7,14,5,7,8,11,14,0。 排序过程如图所示：

![countsort](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/countsort/1.png)

计数排序就是这么简单。但是，上面的排序中有**两个问题**。
1. 辅助数组B浪费了内存空间，可以优化。
2. 这个排序过程不是稳定的。

### 优化：节约内存
上面的排序过程可以这么优化一下：辅助数组的元素个数不必与待排序数组的取值范围一致，可以把待排序数组的最大值和最小值之差（加1）作为辅助数组的元素个数（记住，辅助数组的索引的作用）。

优化后的排序，不再作图演示，直接上代码：
```java
public static void count_sort(int[] array)
{
    int max = array[0];
    int min = array[0];
    for (int var : array)
    {
        if(var > max)
        {
            max = var;
        }
        else if(var < min)
        {
            min = var;
        }
    }

    int size = max - min + 1;
    int[] count_buffer = new int[size];
    for (int var : array)
    {
        count_buffer[var-min] += 1;
    }

    int n = 0;
    for (int i = 0; i < count_buffer.length; ++i) {
        for(int j = 0; j < count_buffer[i]; ++j)
        {
            array[n] = i + min;
            ++n;
        }
    }
}
```

### 优化：稳定排序
稳定排序可以这么做，对上一步的结果中的辅助数组做如下处理：A[i] += A[i-1]; 然后会有如下关系，假设A的某个元素索引为a，num_of_a表示排序后该元素的位置：num_of_a = B[A[a]]，每次取出B的元素值后，就把该元素值减1。

计数调整后的数组元素如图：

![countsort](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/countsort/2.png)

只要逆序遍历待排序数组A，结合上面的表达式，就能得到A的稳定排序的结果。

举例来说，逆序遍历A，第一个元素是0，那么0在排序结果的位置就是B[0] == 1，就是说最后的排序结果中，0是第一个（记得要把B[0]减1）。接着遍历到14，它在排序后的位置是B[14] == 10（记得要把B[14]减1）。再遍历几个元素后又遇到14，它最终的位置是B[14]==13。注意这里，这就是这个计数如何做到稳定排序的。

最终的稳定排序版本的代码：
```java
public static int[] count_sort_stable(int[] array)
{
    int max = array[0];
    int min = array[0];
    for (int var : array)
    {
        if(var > max)
        {
            max = var;
        }
        else if(var < min)
        {
            min = var;
        }
    }

    int size = max - min + 1;
    int[] count_buffer = new int[size];
    for (int var : array)
    {
        count_buffer[var-min] += 1;
    }

    // Transform count_buffer
    
    // After this transformation, the value in count_buffer means sequence of the index of count_buffer
    
    for (int i = 1; i < count_buffer.length; ++i) {
        count_buffer[i] += count_buffer[i-1];
    }

    int[] array_stable_sorted= new int[array.length];
    for (int i = array.length-1; i >= 0; --i) {
        int seq = count_buffer[array[i]-min];
        array_stable_sorted[seq-1] = array[i];
        --count_buffer[array[i]-min];;
    }

    return array_stable_sorted;
}

```

### 总结
从上面的过程中，也可以看出，计数排序的使用场景有限：
1. 它只能对整数排序。
2. 对于待排序数组中最大值和最小值之差很大的情况下，计数排序可能不适合（会使用到很多内存）。

