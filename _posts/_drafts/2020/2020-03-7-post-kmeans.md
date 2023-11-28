---
layout: post
title: "Kmeans clustering"
subtitle: 'Kmeans聚类原理解析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-03-07 17:32 
lang: ch 
catalog: true 
categories: documentation
tags:
  - unsupervised learning
  - Time 2020
---

## Introduction
kmeans是一种无监督聚类算法，本文主要根据CS229 lecture notes进行知识整理，便于以后进行知识回顾。

## Method
假设给定了训练数据集$${x^{(1)}, x^{(2)},\cdots, x^{(m)}}$$，没有相应的样本类别标签，需要我们将这些数据group into cohesive clusters中。具体算法介绍如下：

`1` 随机初始化k个聚类中心：$$\mu_{1}, \mu_{2}, \dots, \mu_{k} \in \mathbb{R}^{n}$$。 <br>
`2` 重复如下过程直到收敛：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; `2.1` 对于每一个数据集中的样例$x^{(i)}$，计算其所属**聚类类别**(欧式距离)：

$$c^{(i)}=\arg \min _{j}\left\| x^{(i)}-\mu_{j} \right\| ^{2}$$

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; `2.2` 将所有样本所属聚类标记好之后，计算每个聚类的新的**聚类中心(centroid)**，其实就是各个维度坐标数值平均：

$$
\mu_{j}=\frac{\sum_{i=1}^{m} 1\left\{c^{(i)}=j\right\} x^{(i)}}{\sum_{i=1}^{m} 1\left\{c^{(i)}=j\right\}}
$$

下图展示了上述kmeans算法可视化流程：

<center><img src="/img/in-post/kmeans/kmeans_flow.pdf" width="100%"></center>

## Convergence
定义畸变函数(distortion function)如下，这个$J$函数用来衡量聚类在所有样本中的聚类误差(使用欧式距离衡量)：

$$
J(c, \mu)=\sum_{i=1}^{m}\left\|x^{(i)}-\mu_{c^{(i)}}\right\|^{2}
$$

其中，$x^{(i)}$是第i个数据样本，$\mu_{c^{(i)}}$表示第i个样本所属的类别的聚类中心。kmeans算法的优化收敛流程可以看作对上面定义的损失函数进行最优化求解。

关于算法的是否能够收敛，引用原文：
> Specifically, the inner-loop of k-means repeatedly minimizes J with respect to c while holding μ fixed, and then minimizes J with respect to μ while holding c fixed. <br>
> Thus, J must monotonically(单调的) decrease, and the value of J must converge. <br>
> Usually, this implies that c and μ will converge too. But there do exist a few different values for c and/or μ that have exactly the same value of J, but this almost never happens in practice.

关于算法保证收敛到局部最小值，引用原文：
> The distortion function J is a non-convex function, and so coordinate descent on J is not guaranteed to converge to the global minimum.

解决收敛到局部的难题的一种方式是，采用多种初始化的`cluster centroid`数值，并且选择最终收敛得到的J函数数值最低的$J(c,\mu)$作为最终结果。

## Set K
kmeans算法需要确定数据划分的聚类数量k，大致有如下几种方式供参考：

`1` 通过数据可视化或者其他分析方式获取数据的先验知识，帮助确定k。<br>
`2` 定义关于k的函数，通过求函数极值产生最佳的k值：`gap statistics`(Estimating the number of clusters in a data set via the gap statistic, Tibshirani, Walther, and Hastie 2001) -> 最大gap statistic对应的k即为所求、`jump Statistic`(finding the number of clusters in a data set, Sugar and James 2003) <br>
`3` 基于结构的算法：即比较类内距离、类间距离以确定k。使用平均轮廓系数(silhouette coefficient)，越趋近1聚类效果越好。使用类内距离、类间距离，其数值越小越好。 <br>
`4` The elbow method。通过绘制横轴为k(from 1 to num_of_samples)，纵轴为畸变函数的折线图，折线首次出现`肘点`的位置对应的k值是最佳的k。 <br>
`5` BIC或AIC值综合判断方式。BIC或AIC值越小(同时最小更好)，对应的k值即为所求。 <br>
`6` 通过可视化层次聚类的方式对于kmeans聚类所需的k进行确定。 <br>

## Pros and Cons
`Pros:` kmeans算法具有原理简单，实现容易，可解释性较强的特点，如果数据量级很大情况下，收敛慢，但是相比大多数聚类算法，仍然就有较好的收敛效率。<br>
`Cons:` kmeans算法中$k$是需要确定的超参，$k$值的选定是非常难以估计的。kmeans算法中通常只能收敛到局部最优。kmeans算法对噪声和离群点敏感(离群值敏感) -> 可以通过LOF离群点检测等算法预先去除离群点。kmeans算法对于非凸簇难以建模(通常只能发现球形簇) -> 这种情况可以通过加kernel的方式通过数据空间转化的方式加以解决，但是仍然需要额外的kernel调试。

## Reference
> CS229 lecture notes - Andrew Ng <br>
> http://sofasofa.io/forum_main_post.php?postid=1000282 <br>
> https://www.zhihu.com/question/29208148 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内