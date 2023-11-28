---
layout: post
title: "MSE MAE Huber Logcosh Quantile"
subtitle: '用于回归的损失函数知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-03-18 10:43
lang: ch 
catalog: true 
categories: documentation
tags:
  - Loss Function 
  - Time 2020
---

## Introduction
本文主要对于用于回归问题的损失函数进行整理，简单分析其特点、应用特性，便于以后知识回顾。本文部分内容根据[这个链接](https://nbviewer.jupyter.org/github/groverpr/Machine-Learning/blob/master/notebooks/05_Loss_Functions.ipynb#)进行整理。

损失函数是机器学习、深度学习乃至一般最优化问题中最重要的部分，损失函数代表了整体模型参数优化的最终目标。同时，对于不同类型的模型以及优化算法来说，其对于损失函数也有一定的要求。

## MSE (L2)
Mean Square Error均方误差，也可以称之为平方损失，或者L2损失，其公式如下：

$$
MSE = \sum\limits_{i=1}^n  {(y_i^{label} - y_i^{predict})}^2
$$

```python
def mse(true, pred):
    return np.sum((true - pred)**2)
```

通过图来了解其性质，误差位于0附近的时候，损失数值平滑可导，但是对于误差非常大的样本点(极有可能是异常点)的损失非常大，那么将会产生错误的参数更新方向：

<center><img src="/img/in-post/loss_function/mse.pdf" width="60%"></center>

## MAE (L1)
Mean Absolute Error平均绝对误差，是目标值和预测值之差的绝对值之和，其公式如下所示：

$$
MAE = \sum\limits_{i=1}^n  {|y_i^{label} - y_i^{predict}|}
$$

```python
def mae(true, pred):
    return np.sum(np.abs(true - pred))
```

<center><img src="/img/in-post/loss_function/mae.pdf" width="70%"></center>

MSE对于异常值`过分的敏感`，将导致如果数据中出现了一个噪声值，那么梯度更新的方向将会受到严重的影像。但是，MAE对于异常值通常被认为是`过分的不敏感`。这两种情况都不是最理想的状态，因此，

## Huber (L1+L2)
Huber损失相比于平方损失来说对于异常值不敏感，但它同样保持了可导的特性。它结合了mae和mse两者的优点，集合了对于异常值不敏感的mae以及平滑处处可导的mse的特性。其公式如下所示：

$$
L_{\delta}(y, f(x))=\left\{\begin{array}{ll}\frac{1}{2}(y_i^{label}-y_i^{predict})^{2} & \text { for }|y_i^{label}-y_i^{predict}| \leq \delta \\ \delta|y_i^{label}-y_i^{predict}|-\frac{1}{2} \delta^{2} & \text { otherwise }\end{array}\right.
$$

```python
def sm_mae(true, pred, delta):
    loss = np.where(np.abs(true-pred)<delta , 0.5*((true-pred)**2), delta*np.abs(true - pred) - 0.5*(delta**2))
    return np.sum(loss)
```

上式表明，当残差大于$\delta$时使用L1损失，否则使用L2损失。因此，huber损失对于$\delta$的选择十分重要。

<center><img src="/img/in-post/loss_function/huber.pdf" width="70%"></center>


## Log-cosh
log-cosh损失函数是一种比L2为平滑的损失函数，利用双曲余弦计算预测误差：

$$
L(y, y^p) = \sum\limits_{i=1}^n  {\log(\cosh(y_i^{label}-y_i^{predict}))}
$$

```python
def logcosh(true, pred):
    loss = np.log(np.cosh(pred - true))
    return np.sum(loss)
```

log-cosh函数的特点是，既可以具备L2损失的优点，也不会受到异常残差数值的太多影像。拥有huber损失的所有优点，与此同时在每一个点都是二阶可导，这对于很多机器学习模型中十分重要，比如使用牛顿法(利用Hessian矩阵)优化梯度的xgboost算法必须需要损失函数能够计算二阶导数。

<center><img src="/img/in-post/loss_function/logcosh.pdf" width="70%"></center>

但是logcosh损失并不是完美无缺的，它还是会在很大误差的情况下梯度和hessian变成了常数，因此对于一些较难优化的问题而言，会出现收敛较慢的现象。

## Quantile loss
在大多数真实世界的预测问题中，我们常常希望看到我们预测结果的不确定性，如股票的预测问题，太过于精确的预测数字往往没有实际意义，通过预测明天股价的取值区间而不是一个具体的数字对于实际操作来说可能更具意义。分位数损失恰好满足这类需要，在需要预测结果是一个取值区间时特别适用。

$$
L_\gamma(y, y^p) = \sum\limits_{i=y_i< y_i^p} ({\gamma-1}).|y_i^{label}- y_i^{predict}| + \sum\limits_{i=y_i\geq y_i^p}  ({\gamma}).|y_i^{label} - y_i^{predict}|
$$

```python
def quantile(true, pred, theta):
    loss = np.where(true >= pred, theta*(np.abs(true-pred)), (1-theta)*(np.abs(true-pred)))
    return np.sum(loss)
```

分位数回归就是平均绝对误差的一种拓展，分位数的数值选取取决于回归问题中对于`正误差`以及`负误差`两者对于总体loss的贡献大小的均衡，即总体loss更多取决于正的$y_i^{label}>y_i^{predict}$还是取决于$y_i^{label}\leq y_i^{predict}$。

如下图所示，左边两幅图展示了不同方差分布的样本散点图，左边图方差为常数，因此，使用ols最小二乘法即可得到非常好的拟合结果，但是对于样本分布方差变化的情况，通过ols难以全面的拟合出其整体分布，通过设置不同的$\gamma$参数的分位数损失，可以预测样本数据区间如右图所示。

<center><img src="/img/in-post/loss_function/quantile.pdf" width="100%"></center>

我们可以利用分位数损失函数来计算出神经网络或者树状模型的区间。下图是计算出基于梯度提升树回归器[gbdt](/documentation/2020/03/16/post-gbdt/)的取值区间，其中上下边界分别使用$\gamma=0.95$以及$\gamma=0.05$计算得到：

<center><img src="/img/in-post/loss_function/quantile1.pdf" width="80%"></center>

## Reference
> https://www.jianshu.com/p/b715888f079b <br>
> https://nbviewer.jupyter.org/github/groverpr/Machine-Learning/blob/master/notebooks/05_Loss_Functions.ipynb#Regression-loss

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内