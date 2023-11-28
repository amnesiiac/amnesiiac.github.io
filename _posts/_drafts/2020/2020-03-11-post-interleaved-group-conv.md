---
layout: post
title: "Interleaved group convolution"
subtitle: 'IGC-conv & group conv原理解析|图解'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-03-11 21:32
lang: ch 
catalog: true 
categories: documentation
tags:
  - conv 
  - Time 2020
---

## Introduction
本文将对Interleaved Group Convolutions for Deep Neural Networks这篇[论文](https://arxiv.org/abs/1707.02725)进行解析，简单介绍group conv(GC)以及interleaved group conv(IGC)的结构细节，并对其特性特点进行简要分析，便于以后进行应用。

## GC vs IGC 
下图中展示了`group conv`卷积的基本流程和`regular conv`的对比。左边流程为普通卷积，右边对应的结构为组卷积。组卷积相当于对于输入特征图按照不同的channel分成一定数量(e.g. 2)的组别，然后分组分别进行卷积，最终得到和正常卷积相同维度的输出。这种结构相当于是一种网络结构模型正则。

<center><img src="/img/in-post/interleaved_group_conv/group_conv.pdf" width="100%"></center>

根据上面的图中所示，可以很容易的发现，组卷积所需要的卷积核参数量为正常卷积的一半，但是整个卷积结构的输入特征图和输出特征图的数据结构不变。<br>
根据上面图示，可以对于原文中的内容进行简单的解释：
> It is known that a group convolution is equivalent to a regular convolution with sparse kernels... <br>
> `explain:` 上述group-conv可以看成是一种使用了稀疏卷积核的正常卷积，这是因为，假设左边的正常卷积中的`a3 & a4`、`b3 & b4`、`c3 & c4`以及`d3 & d4`中的元素数值都为0，那么正常卷积得到的输出将和组卷积相同。

## Interleaved-conv
下面的图展示了Interleaved GC结构的一种具体实现。

<center><img src="/img/in-post/interleaved_group_conv/interleaved_group_conv.pdf" width="80%"></center>

**Wider than regular convolutions** <br>
一般而言，卷积操作的宽度为输入特征图的的通道数，根据原文内容，可以定义$M\times L$作为整个IGC结构的宽度。原文中公式7、8、9证明了，在相同的卷积操作参数复杂度的条件下，IGC卷积结构能够相比regular-conv达到更大的宽度。而该论文的作者后续的实验表明，越宽的网络结构在模型性能上越强。于此同时，该论文表示，较深的网络有可能出现难以训练的问题，即便采用了residual架构也通常不能够完全解决梯度反传消失的问题，因此将模型复杂度转化为横向延伸能够有效的提升效果。

**When is the widest?** <br>
根据论文中公式10-15中的得出，当$L=M\times S$时，获得最大的宽度。其中S的定义：kernel size in the primary group convolution is S。

**Wider leads to better performance?** <br>
根据论文中的结果，这个问题的答案是肯定的。
> The fig suggest that an IGC block near the greatest width, e.g., M = 2 in the two example cases in Figure 3, achieves the best performance.

<center><img src="/img/in-post/interleaved_group_conv/width_result.pdf" width="80%"></center>

## Special cases
**Group-conv to depthwise conv** <br>
如果group-conv分组数=输入特征图的通道数=输出特征图的通道数，则组卷积等效于[depthwise conv](/documentation/2020/01/12/post-separable-conv/)，相关的应用如mobilenet以及Xception。

**Group-conv to Global Depthwise conv** <br>
如果group-conv分组数=输入特征图的通道数=输出特征图的通道数，并且卷积核心的尺寸和输入特征图的尺度相同，则组卷积等效于`global depthwise conv`，该结构在[mobilefacenet](https://arxiv.org/abs/1804.07573)被提出。

## Reference
> https://arxiv.org/abs/1707.02725
> https://www.cnblogs.com/shine-lee/p/10243114.html#scroller-2

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内