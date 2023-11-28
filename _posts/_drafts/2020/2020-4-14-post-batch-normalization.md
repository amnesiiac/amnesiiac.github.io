---
layout: post
title: "Batch Normalization"
subtitle: '基本原理简介及使用方法'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-04-14 17:44 
lang: ch 
catalog: true 
categories: documentation
tags:
  - conv 
  - Time 2020
---
## Abstract
本主要对于神经网络中`batch normalization`相关知识进行整理，文章中的内容主要根据[batch normalization论文](https://arxiv.org/abs/1502.03167)以及[这篇博客](https://www.cnblogs.com/guoyaohua/p/8724433.html)进行归纳总结，主要目的是便于知识回顾。

## Introduction
机器学习领域有一个重要的假设:`IID独立同分布假设`，即假设训练数据和测试数据保持相同的分布(经验分布和真实分布相同)，这个假设是使用训练数据集训练得到的模型能够在测试数据集中有良好表现的基本保障。`batch normalization`即在神经网络训练过程中，保持每一层之间有近似的分布(减少层与层之间internal covariate shift)。

<center><img src="/img/in-post/batch_normalization/variance.pdf" width="90%"></center>

有研究表明在图像处理中对于输入图像进行`whiten`操作(即将输入图像中每个维度特征变换成0均值1方差的正态分布)，能够加速神经网络收敛。因此bn的基本想法是:神经网络的每个hidden layer都是下一个隐层的输入，如果将每一层都做一次`whiten`，应该能够加速模型收敛。

## BN
通过下面的图像对于`bn`的基本想法进行简单解释:

<center><img src="/img/in-post/batch_normalization/bn_activate.pdf" width="90%"></center>

使用`bn`和不使用`bn`的在计算上的简明区别如下图(使用了$conv+bn+activation func$结构):

<center><img src="/img/in-post/batch_normalization/bn.pdf" width="80%"></center>

### BN for train 
下面主要介绍模型训练过程中的`bn`流程。目前主要使用基于mini-batch的优化方式对于`bn`中的非学习参数`mean`和`std`进行计算，以及对于`bn`中的学习参数`gamma`和`beta`进行学习。

<center><img src="/img/in-post/batch_normalization/train_bn.pdf" width="80%"></center>

### BN for infer
下面主要针对模型测试(inference)过程中的`bn`流程进行简介。对于训练中可学习的参数`gamma`和`beta`，直接从训练中保存的数据中得到；但是对于训练中的固定参数`mean`和`beta`，由于有的应用场景下测试时为online inference，单个样本无法计算均值方差。其解决办法是基于IID独立同分布假设，直接使用训练数据集中所有样本的全局均值和方差即可，具体地，在模型训练过程中，记住每一个mini-batch的均值和方差，然后对于这些均值方差求数据期望就得到了我们需要的`infer-mean`和`infer-beta`。

<center><img src="/img/in-post/batch_normalization/test_bn.pdf" width="80%"></center>

## Code
```python
# Method-1 api:tf.nn.moments() & tf.nn.batchnormalization()
import tensorflow as tf

def bn_layer(x,is_training,name='BatchNorm',moving_decay=0.9,eps=1e-5):
    # 获取输入维度并判断是否匹配卷积层(4)或者全连接层(2)
    shape = x.shape
    assert len(shape) in [2,4]
    param_shape = shape[-1]

        # init gamma & beta 
        gamma = tf.get_variable('gamma',param_shape,initializer=tf.constant_initializer(1))
        beta  = tf.get_variable('beat', param_shape,initializer=tf.constant_initializer(0))
        # compute mini-batch mean & var 
        axes = list(range(len(shape)-1))
        batch_mean, batch_var = tf.nn.moments(x,axes,name='moments')
        # using 训练过程中的滑动平均值 update mean & var
        ema = tf.train.ExponentialMovingAverage(moving_decay)

        def mean_var_with_update():
            ema_apply_op = ema.apply([batch_mean,batch_var])
            with tf.control_dependencies([ema_apply_op]):
                return tf.identity(batch_mean), tf.identity(batch_var)

        # when training: update mean & var | when testing: using fixed mean & var in training
        mean, var = tf.cond(tf.equal(is_training,True),mean_var_with_update,
                lambda:(ema.average(batch_mean),ema.average(batch_var)))
        return tf.nn.batch_normalization(x,mean,var,beta,gamma,eps)
```

```python
# Method-2 api: tf.contrib.layers.batch_norm()
import tensorflow as tf

def batch_norm(x,epsilon=1e-5, momentum=0.9,train=True, name="batch_norm"):
    with tf.variable_scope(name):
        epsilon=epsilon, momentum=momentum, name=name
    return tf.contrib.layers.batch_norm(x, decay=momentum, updates_collections=None, epsilon=epsilon,
                                        scale=True, is_training=train,scope=name)
# param: 
# decay: moving average decay rate.
# center: if true has param beta else no param beta. 
# scale: if true has param gamma else no param gamma
# epsilon: 1e-5 default
# activaton_fn: used for activated bn output -> default linear
# param_regularizers: regularizers for gamma & beta
# is_training: when training: 1: use滑动平均后的mean&var 2:save mini-batch mean&var for test
# scope: variable_scope
# updates_collections: 一般使用tf.GraphKeys.UPDATE_OPS ->
update_ops=tf.get_collection(tf.GraphKeys.UPDATE_OPS)
with tf.control_dependencies(update_ops): # todo:需要添加依赖项 
    train_op=optimizer.minimize(loss)
# 当update_collections=None时 设置为强制更新 但是可能导致训练速度减慢 尤其分布式系统
```
## Conclusion
...

## Application
...

## Reference
> https://www.cnblogs.com/guoyaohua/p/8724433.html

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内