---
layout: post
title: "Selective Search"
subtitle: 'region proposal method简析'
author: "twistfatezz"
header-style: text
mathjax: true
date: 2020-05-09 09:29 
lang: ch 
catalog: true 
categories: documentation
tags:
  - imaging principle 
  - Time 2020
---
## Abstract
这篇文章主要对于用于`region proposal`任务的论文selective search中的内容进行整理，便于知识回忆。

## Introduction
图像包含的信息非常丰富，对于图像中的单个物体而言，具有形状(shape)、尺寸(scale)、颜色(color)、纹理(texture)等特征，对于不同的物体而言，具有位置层次关系、以及上述当个图像之间(shape、scale、color、texture)的对比信息。

<center><img src="/img/in-post/selective_search/img_intro.pdf" width="100%"></center>

## Problem
针对上述问题背景，一个好的`region proposal`方法需要具备下面的特征：<br>
**➤ 适应图像中不同尺度的物体(capture all scales)**<br>
一个好的算法能够对于图像中多种尺度多个位置的物体进行区域建议。常见的能够解决上述问题的方法有穷举搜索(`exhaustive selective`)法，对于图像中可能出现物体的所有的位置以及每个位置中的各个尺度进行穷举，但是这种方式非常耗时，因此一些方法采用了在图像上进行网格搜索(`grid search`)以及使用一些易提取的特征(如`HOG`)。`selective search`方法采用图像分割(`img segmentation`)以及使用层次算法(`hierarchical algorithm`)解决这个问题。<br>
**➤ 适应多样化(diversification)** <br>
单一的策略无法应对多种类别任务，因此联合使用颜色、纹理、大小等多种策略对于经过`img segmentation`+`hierarchical algorithm`得到的小区域进行合并。<br>
**➤ 计算速度要快(fast to compute)** <br>
需要设计、优化算法，提高算法运行效率适应应用场景需要。

## Method
### Bottom-up Grouping
这个`selective search`算法是基于小区域基础之上的层次聚类算法产生最终的输出`region proposal`。其中小区域是由[Selective Search for Object Recognition](https://link.springer.com/content/pdf/10.1007/s11263-013-0620-5.pdf)论文中的方法提供的。
### Hierarchical Grouping
<center><img src="/img/in-post/selective_search/hierarchical_grouping.pdf" width="100%"></center>

### Diversification Strategies
为了能够提取得到有效的特征以及能够解决`problem`中的挑战，需要采用多样化的策略进行特征抽取，并且使用多种相似度度量方案进行邻接区域的相似度衡量。最后，区域合并算法中第一个需要进行合并的区域有很多种选择，采用不同的初始合并起点也能够增加算法的鲁棒性。<br>
**Complementary Colour Spaces** <br>
为了使得算法能够适应不同的应用场景以及光照条件，上述层次算法在不同的颜色空间运行，即本文中的算法考虑了在不同的场景（物体所在的背景）下，不同的光照条件下的提取的特征的不变性(invariance)，避免了：改变光照、场景后，猫变成了狗的谬误。`selective search`方法在8个颜色空间上进行，如下表2所示。

<center><img src="/img/in-post/selective_search/color_spaces.pdf" width="80%"></center>

**Complementary Similarity Measures** <br>
在计算区域之间的相似度的时候，为了算法的鲁棒性，采用了4种相似度复合的形式，得到最终的相似度度量。这样做的好处是能够避免使用单一相似度带来的区域划分建议偏差。<br>
`颜色相似度`：对于颜色空间中的三个颜色通道，分别获取其25bins颜色分布直方图，这样得到了75bins的直方图，再进行L1-norm归一化，可以得到每个区域的75维的向量，在通过下面的公式计算出区域$r_i$和$r_j$之间的相似度：

$$s_{color}(r_i,r_j)=\sum_{k=1}^n min(c_i^k,c_j^k)$$

在区域进行合并之后，按照下面的公式计算合并后区域的直方图向量：

$$C_t=\frac{size(r_i)\times C_i+size(r_j)\times C_j}{size(r_i)+size(r_j)},\quad 其中size(r_t)=size(r_i)+size(r_j)$$

上面的公式相当于：按照区域$r_i$和$r_j$面积进行加权求解合并后区域$r_t$的直方图向量。<br>
`纹理相似度`：采用SIFT-like提取得到的特征作为纹理特征。具体地，对于每个颜色通道的8个不同的方向分别计算`gaussian derivatives`($\sigma=1$)，然后经过L1-norm归一化得到10bins直方图，可以用10维度向量进行表示，于是总共得到$8\times 10 \times 3=240$维度特征向量。最后，两个区域之间的纹理相似度可以用下面公式进行计算：

$$s_{texture}(r_i,r_j)=\sum_{k=1}^n min(t_i^k,t_j^k)$$

在区域进行合并之后，按照下面的公式计算合并后区域的直方图向量：

$$T_t=\frac{size(r_i)\times T_i+size(r_j)\times T_j}{size(r_i)+size(r_j)},\quad 其中size(r_t)=size(r_i)+size(r_j)$$

依然是按照合并之前的区域的大小进行加权处理。<br>
`尺寸相似度`：这个相似度特征可以看作用来优先合并小的区域，这样走的目的是：可以有效的避免大的区域不断吞并周围较小的区域，在区域建议上容易出现偏差，因此，对于两个区域越小，区域之间的相似度应当越大。具体地，采用如下公式进行实现：

$$s_{size}(r_i,r_j)=1-\frac{size(r_i)+size(r_j)}{size(im)},\quad 其中size(im)表示整个图像的大小$$

`区域之间形状契合度`：如果区域$r_i$包含在区域$r_j$内部，则应当优先合并，另一方面，如果区域$r_i$很难与区域$r_j$相邻，则不应当将他们合并。定义切合度距离是用来衡量两个区域有多大程度应当合并。其计算方式定义如下：

$$fill(r_i,r_j)=1-\frac{size(BB_{ij})-size(r_i)-size(r_i)}{size(im)},\quad size(BB)指能框住两个区域的最小的bbox的大小$$

`同时考虑上述四种相似度`：采用下面的公式对于上述得到的四种相似度进行综合考虑：

$$s(r_i,r_j)=a_1s_{color}(r_i,r_j)+a_2s_{texture}(r_i,r_j)+a_3s_{size}(r_i,r_j)+a_4s_{fill}(r_i,r_j),\quad 其中a_i\in \{0,1\}$$

$a_i$用来控制是否使用某种相似度，论文中指出，并不通过$a_i$对于不同的特征权重进行衡量。

**Complementary Starting Regions** <br>
论文提到使用不同的初始合并区域对于算法的鲁棒性会有提升。但是，考虑到产生小区域的算法[论文]()已经产生了很好的初始区域合并建议，并且使用不同的颜色空间得到初始化区域建议已经有所不同，因此本论文没有进一步在这方面做工作。

### Combinging Locations
通过上述的合并步骤我们能够得到很多很多的区域，但是每个区域作为目标的可能性都不同，因此我们需要对于每个初始合并得到的region proposal进行打分，取`top-k`即可得到相应的最优的k个含目标区域的建议。<br>
按照`hierarchical grouping`中的顺序给予不同层次得到的区域以打分，其中最后合并得到的完整图像权重为1，前一张图像为2，最先合并的区域权重越大。但是，这样会产生一个问题，由于我们在`hierarchical grouping`的过程中，采用的策略非常多，因此得到的不同层次区域的权重序列也是不同的，这样会产生较多的区域权重相同。对于不同的策略得到的相同权重的区域建议，采用随机数的方式将这些tiers进行区分。<br>
用公式进行表述，假设`grouping strategy`为$j$，则$v_i^j$表示在策略为$j$的条件下，在层次聚类的第$i$轮次(意味着所有策略下，位于第$i$轮次区域有相同的打分)，其中打分都为v_i，则需要通过下式对于它们进行区分：

$$v_i^j=RND\times i,\quad 其中RND是一个[0,1]之间的随机数$$

那么，针对不同的策略得到的原本相同的打分的区域，就可以按照新的打分方式区分它们。相同的区域如果被多个策略所合并出来，那么将它们的权重进行叠加，即多种策略都认为此区域为目标区域，得分应当提高。

## Object Recognition
论文提到了两种可以用于提取目标检测任务相关特征的方法：HOG(histogram of oriented gradients)以及bag-of-words方法。从计算可行角度考虑，使用HOG特征结合linear classifier是仅有的可行的方案，但是HOG往往需要搭配exhaustive search的方式进行实施。因此本论文采用了bag-of-words方案用于目标识别任务。进一步地，本文对于bag-of-words中的特征提取方式进行了改善：使用了SIFT、two-color SIFTs、extended opponent SIFT以及RGB-SIFT方法进行了多尺度金字塔特征提取。图像金字塔共降采样4个层次：$1\times 1$，$2\times 2$，$3\times 3$，$4\times 4$，每个层次使用$\sigma=1.2$进行高斯模糊。最终可以得到长度为360000的特征向量。

其中，本文的目标识别的基本框架十分简单，如下图所示，注意，本文在训练的过程中，使用了hard negative mining的技巧。

<center><img src="/img/in-post/selective_search/pipeline.pdf" width="100%"></center>


## Reference
> Selective Search for Object Recognition <br>
>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
