---
layout: post
title: "Conditional random fields"
subtitle: '原理举例解析|简单应用'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-09-26 10:09 
lang: ch 
catalog: true
categories: documentation 
tags:
  - graphical model
  - Time 2019
---
## Introduction
图是由节点以及连接节点的边组成的集合，节点和边分别记作$v$和$e$，节点和边的集合分别记作$V$和$E$，图模型记作$G=(V,E)$。概率图模型是由图表示的概率分布，其中节点表示随机变量，边表示随机变量之间的依赖关系。当概率图模型所表示的联合概率分布满足成对、局部或者全局马尔可夫性时，就成此联合概率分布为概率无向图模型。关于图模型的概况简述如下图所示：

<img src="/img/in-post/crf/graphic_model.png" width="60%">

## Preparations
「**成对、局部以及全局马尔可夫性质**」这三个性质是等价的关系，也就是说满足任意一个条件能够推出另外两个条件。用一句话概括这个性质对图具体做的约束：对一个节点，在给定他所连接的所有节点的前提下，他与外界是独立的。就是说如果你观测到了所有和这个节点直接连接的那些节点的值的话，那这个节点和那些不直接连接他的点就是独立的。更加精确的理解详见`统计学系方法p191-192`。

「**团和最大团**」无向图中任何两个节点均有边连接的节点的子集合称为团。如果$C$是无向图$G$中的一个团，并且不能再加进$G$中的任何一个节点时期称为一个更大的团，则称$C$为最大团。

「**概率无向图的因子分解**」为什么要定义团的概念呢？是因为在马尔可夫性质的框架下，图模型的联合概率分布$P(Y)$可以分解为图中所有最大团$C$上的函数$\Psi(T_c)$的乘积的形式。具体的：

$$
P(Y)=\frac{1}{Z} \prod_{C} \Psi_{C}\left(Y_{C}\right)
$$

其中：
$$
Z=\sum_{Y} \prod_{C} \Psi_{C}\left(Y_{C}\right)
$$
是归一化因子，$\Psi_c(Y_c)$是$C$上定义的严格正函数(这是为了能够让上面的式子满足概率分布公式的形式)，如果应用在图像分割领域则称之为势函数，如果应用到词性标注方面，称为特征函数。通常取势函数(特征函数)为指数函数：$$\Psi_c(Y_c)=exp \left\{ -E(Y_c)\right\}$$。关于$-E(Y_c)$，需要我们根据具体问题进行定义，本文后面example部分给出了用于词性标注和图像分割的函数的定义。

「**条件随机场**」上面铺垫了这么多，现在终于可以`平滑地`引出条件随机场了。条件随机场是给定了随机变量$X$的条件下，随机变量$Y$的马尔可夫随机场。条件随机场，顾名思义，就是一个条件分布$P(Y\mid X)$满足上面的构成了一概率无向图模型呗。用`词性标注`这个例子对于CRF的定义进行进一步的理解：所需要学习的条件随机场首先是一个条件概率分布$P(Y\mid X)$，其中$Y$是输出变量，表示词性的标记序列，$X$是输入变量，表示需要标注的观测序列。也就是说，给定一堆训练数据，数据中包含若干的单词的词性标签$y$和单词$x$，我们希望通过训练得到一个从单词到单词词性标签的模型，因此CRF是一个判别模型。进一步讲，我们上面对于这个模型加上的约束条件(要求表征这个从单词到单词的词性的条件分布符合马尔可夫性质，并且可以通过模型展开成势函数(特征函数)的因子分解的形式)就是我们的`假设空间`。我们接触的所有的模型都有其`假设空间(局限性)`，数学上通常会用`不妨`来描述这种哲♂学，这是因为，万能的模型，模型中的上帝还没有被人们发现。学习时，利用训练数据通过极大似然估计或者正则化的极大似然估计得到条件概率模型$\hat{P}(Y\mid X)$；预测时，对于给定的输入序列$x$，求出条件概率最大的输出序列$\hat{P}(y\mid x)$。

## Examples
「**词性标注问题**」
输入序列$x$："Bob drank coffee at Starbucks"，输入标注序列$y$："Bob(n) drank(v) coffee(n) at(prep) Starbucks(n)"。那么我们如何根据这个输入，获得从单词到词性的线性链CRF模型呢($P(Y\mid X$)？对于这句话有很多种标注的结果，如"n,v, n, prep, n"或"n ,v ,v ,prep ,n"是分别两种随机标注的结果，那么如果判断那种标注最正确呢？先给出序列标注的CRF模型如下图所示：

<img src="/img/in-post/crf/potential_func.png" width="60%">

对于crf来说，状态转移函数只依赖于包含该节点的最大团的位置的状态，和最大团之外的位置的状态独立(无关)，这给了我们定义特征函数时的一种约束：特征函数的定义域需要在最大团的内部，且需要根据具体问题来定义，以保证特征函数的合理性。对于线性链crf状态转移函数$s$只依赖于当前位置和前一个位置的状态(词性)，上图中横向的虚线框是一个最大团。而$s$是定义在当前位置(节点)上的状态函数，只和当前位置的状态有关。对于$t$和$s$分别给出具体定义如下：

$$
t(y_{i-1},y_{i},x,i) \quad s(y_i,x,i)
$$

$y_i$和$y_{i-1}$是当前位置和前一个位置的状态(词性)，$x=(x_1,x_2,x_3,x_4,x_5)$是输入，$i\in 1-5$表示位置序号。举例使得状态函数更加生动形象：在词性标准任务中，两个动词相连基本不可能，因此我们给负分(构建状态函数的时候随意正负，最终都会被类似softmax的操作给映射成概率)：$t(y_2=verb,y_3=verb,x,i)=-1$；把coffee注释成名次给予正分：$s(y_3=noun,x,i)=1$。我们根据相邻的词语之间的语言构建规则以及当前词语的标注，构建了一系列的特征函数，并且赋予不同特征函数以不同的权重，则可以将线性链crf参数化定义为：

$$
P(y\mid x)=\frac{1}{Z(x)} \exp \left(\sum_{i, k} \lambda_{k} t_{k}\left(y_{i-1}, y_{i}, x, i\right)+\sum_{i, l} \mu_{l} s_{l}\left(y_{i}, x, i\right)\right)
$$

$$
Z(x)=\sum_{y} \exp \left(\sum_{i, k} \lambda_{k} t_{k}\left(y_{i-1}, y_{i}, x, i\right)+\sum_{i, l} \mu_{l} s_{l}\left(y_{i}, x, i\right)\right)
$$

其中$\lambda_k$和$\mu_l$是不同特征函数的权重系数，$Z$函数是归一化因子。如果将转移特征和状态特征统一写成$F(y,x)$后，上述模型可以简写成：

$$
{P_{w}(y\mid x)=\frac{1}{Z_{W}(x)} \exp (w \cdot F(y, x))} 
$$

$$
{Z_{w}(x)=\sum_{y} \exp (w \cdot F(y, x))}
$$

对于给定的$(x,y)$，满足的特征函数越多，模型认为$P_w(y\mid x)$越大。下面如果需要探究使用什么优化算法能够实现$max P(Y\mid X)$，请移步`统计学习方法p199`

Reference中[这篇文章](https://zhuanlan.zhihu.com/p/34261803)对于naive bayes，logistic regression，HMM，线性链crf之间的关系整理的非常好，没有仔细看，以后有时间需要深入理解并补充(/crf/relation.png)。现引用部分观点如下：
> 1 逻辑回归模型(最大熵模型)统计的是训练集中的各种数据满足特征函数的频数(conditional)，而贝叶斯模型统计的是训练集中的各种数据的频数。<br>
> 2 逻辑回归模型(最大熵模型)统计的是训练集中的各种数据满足特征函数的频数，而CRF统计的是训练集中相关数据 (比如说相邻的词，不相邻的词不统计) 满足特征函数的频数。


「**图像分割问题**」
关于图像分割，一般使用Full-connected CRF进行post-processing。关于`CRF for segmantic segmantation`的keras实现可以参考reference中的repo。另外有一个python的crf包可以调用：
```python
import pydensecrf.densecrf as dcrf
```

## Reference
> 1 李航《统计学习方法》<br>
> 2 https://www.jianshu.com/p/55755fc649b1 <br>
> 3 黄靖文的回答 https://www.zhihu.com/question/35866596?sort=created <br>
> 4 https://zhuanlan.zhihu.com/p/34261803 <br>
> 5 https://github.com/twistfatezz/FCN-for-Semantic-Segmentation <br>
> 6 https://github.com/twistfatezz/pydensecrf <br>

> 1 当使用inline数学公式的时候 应当使用`$...$`符号<br>
> 2 当希望数学公式单独成行的时候 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内