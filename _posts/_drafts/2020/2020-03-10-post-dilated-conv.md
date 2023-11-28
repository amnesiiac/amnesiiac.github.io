---
layout: post
title: "Dilated convolution"
subtitle: 'atrous conv原理解析|图解'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-03-10 20:28 
lang: ch 
catalog: true 
categories: documentation
tags:
  - conv 
  - Time 2020
---

## Introduction
dilated convolution(atrous conv)称之为空洞卷积，是一种能够以普通卷积相同的参数量获取更大的输出关于输入图像的感受野的卷积方式。该结构在[这篇论文](https://arxiv.org/abs/1511.07122)中被提出，

<center><img src="/img/in-post/atrous_conv/atrous_conv.pdf" width="60%"></center>

如上图所示，图中红色的圆点表示kernel具体数值，图中的用蓝绿色标记出的正方形区域表示相应的卷积kernel的patch，patch中没有被红色圆点的小格子表示padding的0值。卷积神经网络中的patch表示kernel在输入图像中相应的作用区域。atrous_conv的ratio参数能够反应当前空洞卷积的空洞大小或者kernel参数的稀疏程度。

易知，ratio=1时，空洞卷积和标准卷积没有差别，当ratio>1时，空洞卷积将在相邻的参数之间产生ratio-1的空洞。

## Code
```python
tf.nn.atrous_conv2d(value, filters, rate, padding, name=None)
'''
value:    A 4-D Tensor of type float. It needs to be in the default "NHWC" format. 
filters:    A 4-D Tensor: [filter_height, filter_width, in_channels, out_channels]. 
rate:    A positive int32. rate-1 zeros to insert along consecutive eles.
padding:    A string, either 'VALID' or 'SAME'.
name:    Optional name for the returned tensor.
'''
```

即上述tensorflow实现中，filters参数：[filter_height, filter_width, in_channels, out_channels]仍然填写`真实的`参数维度，这个函数会自动将filter通过insert zero的方式将filter变成空洞的形式。需要注意的是：这种实现方式和上述论文中的方式稍有不同，这种方式填充的kernel考虑了工程优化，将论文中蓝绿色patch中`无关紧要的`边缘padding的零值去掉了。两者的对照可通过下图进行表示：

<center><img src="/img/in-post/atrous_conv/atrous_conv1.pdf" width="80%"></center> 

## Reference
> Multi-Scale Context Aggregation by dilated convolutions.

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内