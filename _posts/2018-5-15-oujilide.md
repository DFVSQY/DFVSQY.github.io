---
layout: post
title: 欧几里得算法
tags:
    - 算法
author: 东方VS清扬
---

欧几里得算法又称辗转相除法，是用于计算两个正整数的最大公约数的经典算法

问题描述：`有整数m,n,其中m > n >= 0,求解m和n的最大公约数？`

基本思想：`设函数 _gcd(m,n)_ 用于求解m和n的最大公约数，则 _gcd(m,n) = gcd(n,m mod n)_.`

##### 算法编写：
~~~c#
static int Gcd(int m, int n)
{
    if (m == n) return m;

    int max = m > n ? m : n;
    int min = m > n ? n : m;
    if (min == 0) return max;

    int r = m % n;
    return Gcd(n, r);
}
~~~

##### 结果测试：
~~~c#
static void Main(string[] args)
{
    int m = 40, n = 56;

    // output is 8
    Console.WriteLine(Gcd(m, n));
}
~~~

