---
layout: post
title: "dynamic programming (DP)"
subtitle: '[数据结构与算法] - 动态规划算法'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-04 21:00
lang: ch 
catalog: true 
categories: algorithm
tags:
  - Time 2021
---
### 一个简单例子
你有三种硬币，面值分别是2元、5元和7元，每种硬币都足够多。买一本书需要27元。如何使用最少的硬币正好付清，无需找钱?

### 确定状态
求解动态规划时，需要开一个数组，如`a[i][j]`，这里状态指的是数组中每个维度代表什么。如何确定状态呢？通常我们应该关注问题的最后一步(递归基)，以及去掉最后一步后的子问题(剩余问题)。

**[最后一步(递归基)]**<br>
这个题目中，我们不能直接写出最优策略支付方案，但是我们知道最优策略一定可以写成$a_1+a_2+a_3...+a_k=27$的形式。这个问题中，最后一步就是最后支付的硬币$a_k$。

**[剩余问题]** <br>
除去最后支付的一枚硬币，剩下的支付就变成了剩余问题($27-a_k$)。我们不关心剩余问题的具体如何求解，我们关心的是剩余问题的求解方式是否和原问题一致。显然，除去最后一枚硬币，剩余问题仍然和原问题同构，因此，满足我们对于递归基-剩余问题划分的基本要求。

<center><img src="/img/in-post/algorithm_img/dp_1.pdf" width="80%"></center>

**[状态转移]** <br>
根据现有的问题划分，我们需要进一步研究原始问题和剩余问题之间的联系，两个问题之间的关系就是`状态转移方程`。不妨假设，$f(k)$表示最少由多少枚硬币拼出k元。于是剩余问题和原始问题可以用下面的三种情况进行关联: <br>

$$
ak=2 => f(k)=f(k-2)+1; \\
ak=5 => f(k)=f(k-5)+1; \\
ak=7 => f(k)=f(k-7)+1; \\
$$

分析出原始问题和同构的剩余问题之间的状态转移关系，则原始问题的求解可以用如下公式进行简化：

$$
f(27)=min(f(27-2), f(27-5), f(27-7))+1;
$$

**[初始情况和边界条件]** <br>
边界条件在求解动态规划问题中非常重要，边界条件界定了动态规划路径的边界，使得问题的求解永远在合理的范围内。<br>
**1 初始条件** 当需要拼出面值为0的时候，$f(0)$应当等于0，根据状态转移方程:$f(0)=f(-2)+1=\infty$，这是不合理的，因此，$f(0)$应当被划入特殊考虑范围。继续验证剩下可能的初始条件：$f(1)=f(1-2)+1=\infty$正确，$f(2)=f(2-2)+1=1$正确...因此本问题的初始条件只有$f(0)$。<br> 
**2 边界情况** 当硬币组合无法拼出27元面值，设置$f(k)=\infty$，即给予边界值以一个`惩罚`，用来限制动态规划的优化方向。`

**[确定计算方向]** <br>
我们已经有了初始情况，那么从初始状态值按照状态转移方程进行推演，就可以得到问题的最终求解。一般的，判断计算顺序正确与否，就是通过确定初始条件后，能够按照状态转移方程进行推演得到目标问题的解答，如果不能，则计算方向是错误的。<br>
本例中，如果我们计算到$f(12)$时，发现$f(12)$计算的依赖项$f(12-2),f(12-5),f(12-7)$都是已经得到的状态，那么这个计算顺序就是正确的，否则就是错误的。

### 总结动态规划求解的步骤
**[一: 划分问题]** <br>
**[二: 找到原问题到剩余问题的转移方程]** <br>
**[三: 确定初始条件以及边界情况]** <br>
**[四: 确定计算顺序]**

### 判断一个问题是否能够使用动态规划解决
**[一: 求最大值最小值问题]** \<左上角到右上角所有路径中最大数字和\>，\<最长上升子序列的长度\> <br>
**[二: 求方案数]** \<有多少种方式走到右下角\>，\<多少种支付方式使得硬币和为sum\> <br>
**[三: 求存在性]** \<取石子游戏，先手是否必胜\>，\<能不能选出k个数使得和为sum\> <br>

## Reference

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
