---
layout: post
title: "Focal Loss"
subtitle: '原理解析|结合论文分析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-10-13 17:39 
lang: ch 
catalog: true 
categories: documentation
tags:
  - Loss Function
  - Time 2019
---

## Introduction
object detection按其流程来说，一般分为两大类。一类是two stage detector([RCNN](/documentation/2019/09/29/post-rcnn/)、[fast-RCNN](/documentation/2019/10/06/post-fast-rcnn/)、[faster-RCNN](/documentation/2019/10/06/post-faster-rcnn/))，另一类则是one stage detector([SSD](/documentation/2019/10/04/post-ssd/)、[yolo v1](/documentation/2019/10/08/post-yolo/)、[yolo v2](/documentation/2019/10/09/post-yolo2/)、[yolo v3](/documentation/2019/10/12/post-yolo3/))。 
虽然one stage detector检测速度可以完爆two stage，但是mAP却不如two stage。 
So Why? <br>
the Reason is: Class Imbalance(正负样本不平衡) 
> one stage detector evaluate 10^4 - 10^5 candidate locations per image, but only a few locations contain objects.

**第一点：**样本中会存在大量的easy examples，且都是负样本(easy negatives)。这样，easy negative会对loss起主要贡献作用，会主导梯度的更新方向。 也因此，网络学不到有用的信息，无法对object进行准确分类。同时，即使舍弃大量的easy negative之后，其实hard negative的数量仍然会多于positive的数量，两者之间仍然需要平衡。

**第二点：**为什么two stage不会有这样的问题呢或者为什么two stage没有one stage这么严重呢？ 
因为，对于two stage的[faster-rcnn](/documentation/2019/10/06/post-faster-rcnn/)来说，首先利用RPN产生region proposal，将产生的region proposal按照`前景`score取`top-K`，这一步就已经删去绝大多数的`easy negative`。然后在根据产生的region proposals和gt计算iou，通过设置iou阈值可以控制正负样本比例为1:3。这样就防止了`hard negative`过多的情况。此外，对于负样本的选取，可以通过在线难例挖掘([OHEM](/documentation/2019/10/13/post-ohem/))，选取有利于网络更新的难分样本，让网络学习到有用的信息，进行参数的更新。

在这个问题的背景下，`focal loss`就诞生了，它的出现是为了解决one-stage的目标检测问题中正负样本不平衡、以及负样本中easy example和hard example比例失衡的两个问题。

## Focal Loss
在介绍focal loss之前，首先通过下面的图像明确几个常用的object detection的概念，可以简单通过图像来理解：

<center><img src="/img/in-post/focal_loss/hard_easy.png" width="100%"></center>

`easy_negative`：负样本特征非常明显，模型比较容易就能够判断成负样本的负样本，设这样样本占一张图像中的比例为$a$。 <br> 
`hard_negative`：作为负样本的特征不如其正样本的特征明显，容易被模型判断成正样本的负样本，设这样的样本占一张图像中的比例为$b$。<br>
`easy_positive`：正样本的特征非常明显，容易被模型判断成正样本的正样本，设这样的样本占一张图像中的比例为$c$。<br>
`hard_positive`：作为正样本的特征不如其负样本的特征明显，容易被模型判断成负样本的正样本，设这样的样本占一张图像中的比例为$d$。

则上述各类样本在一张图像中的比例关系可以近似为：$a>b\approx d>c$。

「**普通的binary cross entropy loss**」

$$
\operatorname{CE}(p, y)=\left\{\begin{array}{ll}{-\log (p)} & {\text {if}\ y=1} \\ {-\log (1-p)} & {\text {otherwise}}\end{array}\right.
$$

可以简写如下：

$$
p_{\mathrm{t}}=\left\{\begin{array}{ll}{p} & {\text {if}\ y=1} \\ {1-p} & {\text {otherwise}}\end{array}\right. 
$$

$$
\mathrm{CE}(p, y)=\mathrm{CE}\left(p_{\mathrm{t}}\right)=-\log \left(p_{\mathrm{t}}\right)
$$

普通的cross entropy loss是focal loss一种特殊的情况，借原论文中的图进行解释(下图中的蓝色曲线就是ce-loss)：

<center><img src="/img/in-post/focal_loss/focal_loss.png" width="100%"></center>

注意，上图中的横轴是`probability of ground truth class`，也就是说，这个$p_t$是：**对于一个输入的正样本，模型输出是正样本的概率，或者，对于一个输入的负样本，模型输出是负样本的概率**。显然，对于$p_t\in (0.6,1)$的那些输入的样本属于well classified example。

> The CE loss can be seen as the blue (top) curve in Figure 1. One notable property of this loss, which can be easily seen in its plot, is that even examples that are easily classified($pt \ge .5$)incur a loss with non-trivial magnitude. When summed over a large number of easy examples, these small loss values can overwhelm the rare class.

根据我对focal loss论文的理解，绘制了下面的图像，通过下图能够对于正样本中的四个概念有着更好的理解：

<center><img src="/img/in-post/focal_loss/pos_neg.png" width="60%"></center>

因此，对于easy classified samples(neg)，以及relative easy classified samples(neg)，这些样本的数量太大，而且它们的loss如果数值很大，那么在bp的时候，就会对于总的梯度产生不好的方向倾向。focal loss的想法就是要抑制这些样本的loss数值以及抑制负样本数量对于loss的影响。

我们为交叉熵中的两项设置一个权重，其中权重因子的大小一般为相反类的比重。即负样本越多，我们给它的权重越小。这样就可以降低负样本对总体loss的影响。定义loss如下：

$$
\mathrm{CE}\left(p_{\mathrm{t}}\right)=-\alpha_{\mathrm{t}} \log \left(p_{\mathrm{t}}\right)
$$

其中对于$\alpha_t$的一个可行的选取是：

$$
\alpha_{t}=\left\{\begin{array}{ll}{\alpha} & {\text { if } y=1} \\ {1-\alpha} & {\text { otherwise }}\end{array}\right.  \quad 其中\alpha \in (0,1)
$$

但是，这只是解决了正负样本的不平衡，但是并没有解决easy和hard examples之间的不平衡。 因此，我们定义loss： 

$$
\mathrm{FL}\left(p_{\mathrm{t}}\right)=-\left(1-p_{\mathrm{t}}\right)^{\gamma} \log \left(p_{\mathrm{t}}\right)
$$

其中，$\gamma\in [1,5]$. 这样，当对于简单样本，$p_t$会比较大，所以权重$(1-p_t)^{\gamma}$自然减小了。针对hard example，$P_t$比较小，则$(1-p_t)^{\gamma}$比较大，让网络倾向于利用这样的样本来进行参数的更新。我们把这两种单独的改进进行合并，最终Focal Loss的形式为：

$$
\mathrm{FL}\left(p_{\mathrm{t}}\right)=-\alpha_{\mathrm{t}}\left(1-p_{\mathrm{t}}\right)^{\gamma} \log \left(p_{\mathrm{t}}\right) \quad \alpha_{t}=\left\{\begin{array}{ll}{\alpha} & {\text { if } y=1} \\ {1-\alpha} & {\text { otherwise }}\end{array}\right. \quad p_{\mathrm{t}}=\left\{\begin{array}{ll}{p} & {\text {if}\ y=1} \\ {1-p} & {\text {otherwise}}\end{array}\right.
$$

可以将$y=1$和$y\neq1$写成一个公式进行表达：

$$
\mathrm{FL}\left(p_{\mathrm{t}}\right)=-\alpha y log^{p_t} - (1-\alpha)(1-y)log(1-p_t), \ 其中 y=1(pos)\ or\ y=0(neg)
$$

这样既做到了解决正负样本不平衡，也做到了解决easy与hard样本不平衡的问题。

## Code
等待整理...

## Reference
> 部分翻译自原文 Focal Loss for Dense Object Detection <br>
> 转载并修改自:https://blog.csdn.net/LeeWanzhi/article/details/80069592

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内