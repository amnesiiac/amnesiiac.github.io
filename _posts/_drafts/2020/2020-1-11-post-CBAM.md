---
layout: post
title: "CBAM"
subtitle: 'Convolutional Block Attention Module'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-01-11 10:32 
lang: ch 
catalog: true
categories: documentation
tags:
  - attention mechanism
  - Time 2020
---
## Abstract
本文要介绍的是[CBAM: Convolutional Block Attention Module](https://arxiv.org/pdf/1807.06521.pdf)。这篇文章主要的工作是将卷积神经网络中的`channel attention`和`spatial attention`整合在一起，形成一种卷积神经网络通用的注意力机制结构。据该论文所称，可以达到比单纯的resnet(backbone)以及仅基于通道注意力机制的网络结构[SEnet](/documentation/2020/01/11/post-SEnet/)有更好的实验效果。

## Model
论文提出的模型架构比较简单，直接引用论文原图。下面展示了论文所提出的结构overview结构图。基本结构非常简单，总体思路是`input fmap`首先经过一个通道注意力加权结构，中间得到`intermediate fmap`经过特征图加权图进行按元素点乘得到一个`通道注意力+空间注意力`的卷积神经网络通用结构。

## Overview
<center><img src="/img/in-post/cbam/overview.pdf" width="60%"></center>

## Code

```python
from __future__ import absolute_import
from __future__ import division
from __future__ import print_function

import tensorflow as tf

def cbam_block(input_feature, name, ratio=8):
  """Contains the implementation of Convolutional Block Attention Module(CBAM) block.
  As described in https://arxiv.org/abs/1807.06521.
  """
  with tf.variable_scope(name):
    attention_feature = channel_attention(input_feature, 'ch_at', ratio)
    attention_feature = spatial_attention(attention_feature, 'sp_at')
    print ("CBAM Hello")
  return attention_feature
```

## Channel attention
<center><img src="/img/in-post/cbam/channel.pdf" width="60%"></center>

## Code
```python
def channel_attention(input_feature, name, ratio=8):
  
  kernel_initializer = tf.contrib.layers.variance_scaling_initializer()
  bias_initializer = tf.constant_initializer(value=0.0)
  
  with tf.variable_scope(name):
    
    channel = input_feature.get_shape()[-1]
    avg_pool = tf.reduce_mean(input_feature, axis=[1,2], keepdims=True)
        
    assert avg_pool.get_shape()[1:] == (1,1,channel)
    avg_pool = tf.layers.dense(inputs=avg_pool,
                                 units=channel//ratio,
                                 activation=tf.nn.relu,
                                 kernel_initializer=kernel_initializer,
                                 bias_initializer=bias_initializer,
                                 name='mlp_0',
                                 reuse=None)   
    assert avg_pool.get_shape()[1:] == (1,1,channel//ratio)
    avg_pool = tf.layers.dense(inputs=avg_pool,
                                 units=channel,                             
                                 kernel_initializer=kernel_initializer,
                                 bias_initializer=bias_initializer,
                                 name='mlp_1',
                                 reuse=None)    
    assert avg_pool.get_shape()[1:] == (1,1,channel)

    max_pool = tf.reduce_max(input_feature, axis=[1,2], keepdims=True)    
    assert max_pool.get_shape()[1:] == (1,1,channel)
    max_pool = tf.layers.dense(inputs=max_pool,
                                 units=channel//ratio,
                                 activation=tf.nn.relu,
                                 name='mlp_0',
                                 reuse=True)   
    assert max_pool.get_shape()[1:] == (1,1,channel//ratio)
    max_pool = tf.layers.dense(inputs=max_pool,
                                 units=channel,                             
                                 name='mlp_1',
                                 reuse=True)  
    assert max_pool.get_shape()[1:] == (1,1,channel)
    scale = tf.sigmoid(avg_pool + max_pool, 'sigmoid')
  return input_feature * scale
```
## Spatial attention
<center><img src="/img/in-post/cbam/spatial.pdf" width="60%"></center>

## Code 
```python
def spatial_attention(input_feature, name):
  kernel_size = 7
  kernel_initializer = tf.contrib.layers.variance_scaling_initializer()
  with tf.variable_scope(name):
    avg_pool = tf.reduce_mean(input_feature, axis=[3], keepdims=True)
    assert avg_pool.get_shape()[-1] == 1
    max_pool = tf.reduce_max(input_feature, axis=[3], keepdims=True)
    assert max_pool.get_shape()[-1] == 1
    concat = tf.concat([avg_pool,max_pool], 3)
    assert concat.get_shape()[-1] == 2
    concat = tf.layers.conv2d(concat,
                              filters=1,
                              kernel_size=[kernel_size,kernel_size],
                              strides=[1,1],
                              padding="same",
                              activation=None,
                              kernel_initializer=kernel_initializer,
                              use_bias=False,
                              name='conv')
    assert concat.get_shape()[-1] == 1
    concat = tf.sigmoid(concat, 'sigmoid')
  return input_feature * concat
```

## Reference
> https://raw.githubusercontent.com/kobiso/CBAM-tensorflow/master/attention_module.py

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内