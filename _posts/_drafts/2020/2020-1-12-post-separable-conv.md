---
layout: post
title: "Separable convolution"
subtitle: 'group conv | depthwise conv | pointwise conv'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-01-12 10:11 
lang: ch 
catalog: true 
categories: documentation
tags:
  - conv 
  - Time 2020
---
## Abstract
本文主要的目的是总结关于可分离卷积的知识内容，方便以后查阅进行知识回忆。主要介绍三个内容:`depthwise conv`和`pointwise conv`以及`group conv`。其中`depthwise conv`和`pointwise conv`可以被统一称作`depthwise separable conv`。


## Regular conv
通过下图对于常规卷积运算原理进行简单介绍。
<center><img src="/img/in-post/deconv/conv.png" width="80%"></center>


## Depthwise conv
`depthwiseconv`整个卷积操作中输入特征图的通道数量与输出特征图的的通道数相同。并且，它对输入特征图的的每个通道的图像分别进行卷积运算，没有有效的利用不同通道在相同空间位置上的融合的`feature`信息。因此需要一般需要利用下一节中介绍的`pointwiseconv`来将`depthwiseconv`得到的特征图进行组合生成融合后的特征图。
<center><img src="/img/in-post/separable_conv/depthwiseconv.pdf" width="80%"></center>

## Code
```python
tf.nn.depthwise_conv2d(input,filter,strides,padding,rate=None,name=None,data_format=None)
""" 
input:  input with shape of [batch, height, width, in_channels]
filter:  4d tensor with shape of [filter_height, filter_width, in_channels, out_channels]
strides:  卷积filter的滑动步长
padding:  string 'SAME' 'VALID'
rate:  这个参数的详细解释见下一篇博文[空洞卷积原理及其实现]
"""
```

## Pointwise conv
`pointwiseconv`的运算可以看成一种特殊的常规卷积运算，是一种卷积核的尺寸为 $1\times 1\times M$，M为`input feature map`的通道数。其中`pointwiseconv`的输出特征图通道数等于卷积核的组数，即卷积核的`output channel`数。其计算原理简介如下图。
<center><img src="/img/in-post/separable_conv/pointwiseconv.pdf" width="80%"></center>

## Code
```python
# pointwise conv是一种filter尺寸为1x1的卷积 直接使用python中的普通的卷积构建相应网络结构即可
```

## Reference

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内