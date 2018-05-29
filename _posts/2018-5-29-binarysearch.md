---
layout: post
title: 二分查找
tags:
    - 算法
author: 东方VS清扬
---

二分查找即折半查找，是一种较高效的经典查找算法，对于被搜寻的序列要求必须已排序，另外由于二分查找需要方便的随机存储，故该方法仅适用于线性表的顺序存储结构，不适合链式存储结构。二分查找时间复杂度为 `O(log2n)`。

#### 基本思想
~~~
1. 将待查找的值与给定序列的中间值作比较，若相等则查找成功。
2. 若不相等，则根据序列的增减性在前半部分或者后半部分以同样方法进行查找，直到查找成功或者待查询序列搜寻完毕。
~~~

#### 代码实现
~~~c#
static int BinarySearch(List<int> sorted, int pattern)
{
    int pos = -1;
    int start = 0, end = sorted.Count - 1;
    while (start <= end)
    {
        int middle = (start + end) / 2;

        if (sorted[middle] == pattern)
        {
            pos = middle;
            break;
        }

        if (sorted[middle] > pattern)
            end = middle - 1;
        else
            start = middle + 1;
    }
    return pos;
}
~~~

#### 结果测试
~~~c#
static void Main(string[] args)
{
    List<int> list = new List<int>()
    {
        -738817,-442472,-365580,-247109,
        -210707,-157678,-77867,77867,85168,
        123414,129801,155091,287381,324110,
        508681,604242,626686,686904,855430,982455
    };

    // output is 
    Console.WriteLine(BinarySearch(list, 77867));

    // output is zero
    Console.WriteLine(BinarySearch(list, 800000));

    Console.WriteLine();
}
~~~