---
layout: post
title: "MLE and MAP - point estimation"
subtitle: '原理举例解析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-09-26 17:44 
lang: ch 
catalog: true
categories: documentation 
tags:
  - parameter estimation
  - statistics theroy 
  - Time 2019
---

### Introduction
最大似然估计(maximum likelihood estimation)和最大后验估(maximum a posteriori)都属于统计学中的点估计。最大似然估计属于“频率派”的思想，最大后验估计属于“贝叶斯派”思想。下面的图片展示了统计学基本理论体系。

<img src="/img/in-post/mle_map/flow.png" width="50%">

### Diff between MLE/MAP
以抛硬币游戏为例。现抛一枚硬币5次，观测结果是一正四反。

**MLE的观点** 估计抛硬币所得到的结果是一个随机变量，想要估计这个随机变量的概率密度函数(0-1分布)的参数p，就是直接以观测到的结果为依据，计算能够最大化的"重现"二正一反这个实验的参数p(p=1/5=0.2)。

**MAP的观点** 抛硬币仅抛5次，试验结果提供的信息显然不够有说服力，因此我们希望通过了解硬币制作的质量好坏(比如硬币制作是否平衡对称等)来获得一个更加有说服力的结果，这样就需要引入先验。<br>
现在我们得到一个信息：这枚硬币是制币场制造，上述参数p一般情况下是0.5。如果我们对于制币厂很信赖，认为制币厂质量报告中p=0.5是经历1000次实验所得到经验结论，那么现在对于p的估计可以写为：$$(0.2*5+0.5*1000)/(1000+5)=0.4985$$。如果我们对于制币厂不信赖，认为报告中写的p=0.5仅经历了5次检验，那么对于的估计可以写成：$$(0.2*5+0.5*5)/(5+5)=0.35$$。

**MLE和MAP的差异** MLE以当前实验观测结果作为基准，以观测结果的频率去代替随机变量的概率分布，并求解使当前观测结果出现的可能性最大的参数。MAP则不仅仅考虑了当前的观测结果，还考虑了随机变量的先验知识，结合两者，最大化后验概率，得到参数的估计。

**MLE和MAP在神经网络中的引申** 另外，标准的神经网络优化可以看作是一种极大似然估计(MLE)，但是很多时候，MLE求出的分布参数不好，如上面举的例子中显然MAP求解的数值更加准确，因此在优化过程中适当引入先验知识，利用最大化后验概率计算参数。这种引入先验知识的想法属于神经网络正则化的范畴。接下来使用数学语言对于MLE和MAP之间的关系进行描述。

### MLE
令$P(X\mid \theta)$表示最大似然函数，那么参数$\theta$可以通过下面的公式求解：

$$
\theta_{MLE}=\underset{\theta}{\arg \max } P(X\mid  \theta)=\underset{\theta}{\arg \max } \prod_{i} P\left(x_{i}\mid \theta\right)
$$

通常情况下，对于很多$x_i$，$P(x_i\mid \theta)$都接近0，这使得上述公式无法计算(computation underflow)。可以在将函数映射到对数空间中进行计算：

$$
\theta_{MLE}=\underset{\theta}{\arg\max} \log P(X\mid \theta)=\underset{\theta}{\arg\max} \log \prod_{i} P\left(x_{i}\mid \theta\right)=\underset{\theta}{\arg\max} \sum_{i} \log P\left(x_{i}\mid \theta\right)
$$

求解上面目标函数的最优化问题可以使用最优化方法如梯度下降进行求解。

### MAP
对于最大后验来说，不仅仅要考虑似然函数，还要考虑随机变量先验分布，根据贝叶斯定律有：

$$
P(\theta\mid X)=\frac{P(X\mid \theta) P(\theta)}{P(X)} \propto P(X\mid \theta) P(\theta)
$$

先验分布$P(X)$可以看成是一个定值常数，因此后验概率分布仅和似然概率以及参数分布相关。同样求解最大化后验概率：

$$
\theta_{MAP}=\underset{\theta}{\arg\max} P(X\mid \theta) P(\theta)\\ =\underset{\theta}{\arg\max} \log P(X\mid \theta)+\log P(\theta)\\ =\underset{\theta}{\arg\max} \log \prod_{i} P\left(x_{i}\mid \theta\right)+\log P(\theta)\\ =\underset{\theta}{\arg \max } \sum_{i} \log P\left(x_{i}\mid \theta\right)+\log P(\theta)
$$

比较MLE和MAP公式中，很容易发现，唯一区别在于$\log P(\theta)$。**如果$P(\theta)$和$\theta$无关，如X$\sim$均匀分布，此时MAP等价于MLE.**

### Reference
> Inspiration from https://wiseodd.github.io/techblog/2017/01/01/mle-vs-map/<br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
