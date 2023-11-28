---
layout: post
title: "Mobilenets"
subtitle: 'Squeeze-and-Excitation Networks'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-01-12 14:51 
lang: ch 
catalog: true 
categories: documentation
tags:
  - attention mechanism
  - Time 2020
---
## Abstract
本文要介绍的是[Squeeze and Excitation Networks](https://arxiv.org/pdf/1709.01507.pdf)。这是2017年的`imagenet`挑战赛分类任务上的冠军模型。

## Model
下面简要的介绍其构建结构思路以及相应的tensorflow实现代码。
### Overview
下图展示了SEnet中`squeeze and excitation`模块的结构框图。思路是通过对于每个通道的`fmap`利用`average pooling`$+$ `2 fc network` $+$ `softmax`实现通道重要性分数提取(是一个权重向量)，然后分别将提取的重要性分数向量和输入特征图进行对应点乘，以对输入特征图各个通道进行加权。
<center><img src="/img/in-post/senet/se.pdf" width="80%"></center>

### Code
```python
from __future__ import absolute_import
from __future__ import division
from __future__ import print_function

import tensorflow as tf

def se_block(input_feature, name, ratio=8):
  """Contains the implementation of Squeeze-and-Excitation block.
  As described in https://arxiv.org/abs/1709.01507.
  """
  kernel_initializer = tf.contrib.layers.variance_scaling_initializer()
  bias_initializer = tf.constant_initializer(value=0.0)
  with tf.variable_scope(name):
    channel = input_feature.get_shape()[-1]
    # global average pooling - by reduce_mean is more efficient!
    squeeze = tf.reduce_mean(input_feature, axis=[1,2], keepdims=True)    
    excitation = tf.layers.dense(inputs=squeeze,
                                 units=channel//ratio,
                                 activation=tf.nn.relu,
                                 kernel_initializer=kernel_initializer,
                                 bias_initializer=bias_initializer,
                                 name='bottleneck_fc')    
    excitation = tf.layers.dense(inputs=excitation,
                                 units=channel,
                                 activation=tf.nn.sigmoid,
                                 kernel_initializer=kernel_initializer,
                                 bias_initializer=bias_initializer,
                                 name='recover_fc')    
    scale = input_feature * excitation    
  return scale
```

### SEnet-inception
下面的结构图展示了`SE module`嵌入到`inception`的方式。
<center><img src="/img/in-post/senet/seinc.pdf" width="40%"></center>

### SEnet-resnet
下面的结构图展示了`SE module`嵌入到`resnet`的方式。
<center><img src="/img/in-post/senet/seres.pdf" width="40%"></center>

### SEnet-variants
下面的结构图展示了，以嵌入到`resnet`为例子，将`SE module`嵌入到其中的几种不同的方式。对于这几种resnet变种方式，作者都做了相应的验证，并且在论文导论部分说明，这几种方式，都能在`imagenet`分类任务上表现良好，并且作者据此说明:`SE module`是可以被当作一种嵌入模型的通用网络模块。
<center><img src="/img/in-post/senet/variants.pdf" width="80%"></center>

## Reference

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内