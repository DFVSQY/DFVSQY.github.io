---
layout: post
title: Stack应用之算术表达式求值
tags:
    - 算法
author: 东方VS清扬
---

算术表达式求值是Stack应用的经典案例，本篇文章用于记录复习该内容。

##### 基本思想：
~~~
1. 建立两个Stack，一个用于保存运算符，一个用于保存操作数
2. 表达式从左至右依次入栈，操作数入操作数栈，运算符入运算符栈，忽略左括号
3. 在遇到右括号时弹出一个运算符，弹出所需数量的操作数，并将操作结果压入操作数栈
4. 表达式计算完毕时将只有操作数栈中剩余一个元素，为最后结果值
~~~

为了不使问题复杂化仅了解其基本核心思想，我们仅考虑加减乘除四则运算，并且算术表达式每步计算均添加括号（无论其优先级）。

##### 代码编写：
~~~c#
static double ParseExpression(string exp)
{
    Stack<char> ops = new Stack<char>();
    Stack<double> vals = new Stack<double>();

    for (int i = 0; i < exp.Length; i++)
    {
        char c = exp[i];

        if (c.Equals('+') || c.Equals('-') || c.Equals('*') || c.Equals('/'))
        {
            ops.Push(c);
        }
        else if (c.Equals(')'))
        {
            char op = ops.Pop();
            double v = vals.Pop();
            if (op.Equals('+')) v = vals.Pop() + v;
            else if (op.Equals('-')) v = vals.Pop() - v;
            else if (op.Equals('*')) v = vals.Pop() * v;
            else if (op.Equals('/')) v = vals.Pop() / v;
            vals.Push(v);
        }
        else if (char.IsNumber(c))
        {
            double.TryParse(c.ToString(), out double v);
            vals.Push(v);
        }
    }

    return vals.Pop();
}
~~~



##### 结果测试:
~~~c#
static void Main(string[] args)
{
    string exp = "(5 * (((3 + 7) / 2) - 2))";

    // output is 15
    Console.WriteLine(ParseExpression(exp));
}
~~~