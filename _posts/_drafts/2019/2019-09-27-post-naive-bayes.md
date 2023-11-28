---
layout: post
title: "Naive Bayes"
subtitle: '原理简要解析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-09-27 10:33 
lang: ch 
catalog: true
categories: documentation 
tags:
  - generative model 
  - statistics theroy 
  - Time 2019
---

## Introduction
朴素贝叶斯法是一种生成模型。对于给定的数据集，首先基于特征条件的独立性，学习输入输出的联合概率分布(生成模型的基本特征)，然后基于此模型，对于给定的输入$x$，利用贝叶斯定理求出后验概率(MAP)最大的输出$y$。

## Basic principles 
首先对于将用到的数学符号进行声明。$x_i$表示输入数据集中的第$i$个样本，$x^{j}$表示对于一个输入的第j个特征($j\in 1,2,...,n$)。$P(Y=c_k)\ k=1,2,...,K$是先验概率分布，$P(Y=c_k\mid X=x)$是后验概率分布。注意先验概率分布和后验概率分布的定义是由问题本身定义的。对于一个输入$x$分类成输出$y$的问题，$p(Y=c_k)$是先验概率，$p(Y=c_k\mid X=x)$是后验概率，然而对于一个参数估计的问题，如介绍[MLE和MAP](/documentation/2019/09/26/post-MLE-and-MAP/)的文章中所用的：$p(\theta)$是先验概率，$p(\theta\mid x)$是后验概率，$p(x\mid \theta)$是极大似然。

朴素贝叶斯的目标是从训练数据$$train=\left\{(x_1,y_1),(x_2,y_2),...(x_n,y_n)\right\}​$$学习得到输入输出的联合分布$P(X,Y)$。但是上述联合分布不易获得，根据：$$P(X,Y)=P(Y\mid X)P(X)$$，我们可以通过分别计算$P(Y\mid X)$和$P(X)$来得到联合概率。其中后验概率可以通过贝叶斯定律进行分解：

$$
P(Y=c_k\mid X=x)=\frac{P(Y=c_k,X=x)}{P(X=x)}=\frac{P(X=x\mid Y=c_K)P(Y=c_k)}{\sum_k \left[ P(X=x\mid Y=c_k)p(Y=c_k) \right]}
$$

但是上面的式子中，条件概率分布$P(X=x\mid Y=c_K)​$在实际中很难计算，根据`统计学习方法p47`中的说明，由于$$P(X=x\mid Y=c_k)=P(X^{(1)}=x^{(1)},X^{(2)}=x^{(2)},...,X^{(n)}=x^{(n)}\mid Y=c_k)$$，且若输入$x_j$中的每一个“特征分量“有$S_j$个，$Y$的取值有$K$个，那么总的参数量为指数量级：$K\prod_{j=1}^{n}S_j$。因此朴素贝叶斯对于上面的联合分布做出了如下的假设，即：输入的各个特征之间针对特定的输出($Y=c_k$)是独立的：

$$
P(X=x\mid Y=c_k)=\prod_{j=1}^{n}P(X^{(j)}=x^{(j)}\mid Y=c_k)
$$

上述`朴素`的假设通过一个例子来感受下：输入是一幅mxn的图像，共有mxn个特征，朴素贝叶斯的假设认为这些特征之间对于图像分类label无关，这显然是一个`很强`的假设，逃)

将上面那个使问题变得`朴素`的假设代入我们需要求解的后验概率公式中得到：

$$
P\left(Y=c_{k}\mid X=x\right)=\frac{P\left(Y=c_{k}\right) \prod_{j} P\left(X^{(j)}=x^{(i)}\mid Y=c_{k}\right)}{\sum_{k} P\left(Y=c_{k}\right) \prod_{j} P\left(X^{(j)}=x^{(j)}\mid Y=c_{k}\right)}, \quad k=1,2, \cdots, K
$$

已经得到了后验概率表达式，想到朴素贝叶斯希望后验概率最大化，因此，模型可以表示如下：

$$
y=f(x)=\underset{c_{k}}{\arg \max} \frac{P\left(Y=c_{k}\right) \prod_{j} P\left(X^{(j)}=x^{(j)}\mid Y=c_{k}\right)}{\sum_{k} P\left(Y=c_{k}\right) \prod_{j} P\left(X^{(j)}=x^{(j)}\mid Y=c_{k}\right)}
$$

注意到，上面的公式中分母和$c_k$无关，所以可以直接去掉，得到最终的形式：

$$
y=\underset{c_k}{\arg \max}P(Y=c_k)\prod_{j}P(X^{(j)}=x^{(j)}\mid Y=c_k)
$$

## Risk minimization
先上结论：朴素贝叶斯估计中，后验概率最大化等价于期望风险最小化。为了将这点讲清楚，首先需要定义下损失函数：
$$
L(Y,f(X))=\left\{\begin{array}{l} {1, \ Y\ne f(X)} \\ {0, \ Y=f(X)} \end{array}\right.
$$

朴素贝叶斯的期望风险和两个东西有关：第一个是损失函数$L$，第二个是后验概率$P(Y=c_k\mid X=x)$所表示的贝叶斯模型。

$$
R_{\exp }(f)=E[L(Y, f(X))]
$$

风险的期望和$Y$的取值以及$X$的取值相关，先按照$Y$的取值进行展开，那么期望风险函数可以写作：

$$
R_{\mathrm{exp}}(f)=E_{X} \sum_{k=1}^{K}\left[L\left(c_{k}, f(X)\right)\right] P\left(c_{k}\mid X\right)
$$

$$
=\arg \min _{y \in \mathcal{Y}} \sum_{k=1}^{K} L\left(c_{k}, y\right) P\left(c_{k}\mid X=x\right)
$$

因为这里没有对$X$的分布作出假设，所以没有必要进一步展开。根据0-1损失的特点，将上式化简得到：

$$
=\arg \min _{y \in \mathcal{Y}} \sum_{k=1}^{K} P\left(y \neq c_{k}\mid X=x\right)
$$

$$
=\arg \min _{y \in \mathcal{Y}}\left(1-P\left(y=c_{k}\mid X=x\right)\right) 
$$

$$
=\arg \max _{y \in \mathcal{Y}}\left(P\left(y=c_{k}\mid X=x\right)\right)
$$

这样就证明了经验损失函数的最小化等价于后验概率的最大化。**一点启发：如果我们遇到优化求解后验概率最大化的问题，如果问题比较难以求解，可以转化成经验损失最小化的问题去优化。**

## Param estimation 
「最大似然形式的贝叶斯模型参数估计」注意到我们的学习的目标是关于输入和输出的联合分布$P(X,Y)$，根据$$P(X,Y)=P(X\mid Y)P(Y)=P(X=x\mid Y=c_k)P(Y=c_k)$$，其中:
$$
P\left(Y=c_{k}\right)=\frac{\sum_{i=1}^{N} I\left(y_{i}=c_{k}\right)}{N}, k=1,2, \cdots, K
$$
对于一个给定的训练集合来说标签($Y$)的分布是一个已知量。因此需要对另一部分(似然函数)进行求解，将$x$展开如下：

$$
P\left(X^{(j)}=a_{j l}\mid Y=c_{k}\right)=\frac{P\left(X^{(j)}=a_{j l}, Y=c_{k}\right)}{P(Y=c_k)}
$$

使用[指示函数](#jump0)对于两个概率进行展开：<span id="return0"> </span>

$$
\begin{array}{l}
{P\left(X^{(j)}=a_{j l}\mid Y=c_{k}\right)=\frac{\sum_{i=1}^{N} I\left(x_{i}^{(j)}=a_{j l}, y_{i}=c_{k}\right)/N}{\sum_{i=1}^{N} I\left(y_{i}=c_{k}\right)/N} }\\
{\quad \small{j=1,2,...,n;\ l=1,2,...S_j;\ k=1,2,...,K;}}
\end{array}
$$

式中，$x_i^{j}$是第$i$个样本的第$j$个特征；$a_{jl}$是第$j$个特征可能取得第$l$个值。现在似然函数中的最重要的部分已经构建完成，接下来只需要按照似然函数的格式去写出似然函数，套用求解似然函数最大的优化算法即可。

「贝叶斯估计」最大似然框架的概率估计公式没有考虑所要估计的概率值为0的情况，这会影响到后验概率的计算结果，采用贝叶斯形式的概率估计公式可以解决这种问题，该形式相当于在最大似然估计的基础上加入一个$\lambda$进行平滑：

$$
P_{\lambda}\left(X^{(j)}=a_{j l}\mid Y=c_{k}\right)=\frac{\sum_{i=1}^{N} I\left(x_{i}^{(j)}=a_{ji}, y_{i}=c_{k}\right)+\lambda}{\sum_{i=1}^{N} I\left(y_{i}=c_{k}\right)+S_{j} \lambda}
$$

容易验证，上面“修正的最大似然概率估计公式”满足概率大于0且概率之和为1，因此“修正后”的概率估计公式仍然是一种概率分布。同样，贝叶斯先验概率公式可以表示为：

$$
P_{\lambda}\left(Y=c_{k}\right)=\frac{\sum_{i=1}^{N} I\left(y_{i}=c_{k}\right)+\lambda}{N+K \lambda}
$$

特别地，上面两个公式如果取$\lambda=1$，叫做拉普拉斯平滑。

「可参考文章」
关于MLE和MAP以及Bayesian的关系可以参考知乎[这篇文章](https://zhuanlan.zhihu.com/p/37215276)
关于贝叶斯估计中的拉普拉斯平滑的推导过程可以参考[这篇文章](https://zhuanlan.zhihu.com/p/24291822)该文章利用贝叶斯共轭先验导出拉普拉斯平滑表达形式。

「补充内容」<span id="jump0">指示函数定义:</span>
$$
I_A(x)=\left\{ \begin{array}{l}{1,\ if\ x\in A}\\ {0,\ if\  x \notin A} \end{array} \right.
$$ [返回](#return0)

## Reference
> 李航《统计学习方法》p47 -> 我这篇文章的介绍顺序相比李航老师的介绍顺序有所调整。我的顺序是由贝叶斯公式入手，加上“朴素”假设，最终得到了朴素贝叶斯模型的一般形式，而李航老师的是由“朴素”假设开始，再借助贝叶斯公式进行求解。李航老师的顺序是按照朴素贝叶斯算法的计算流程展开的，而我的顺序则更加便于读者理解。

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内