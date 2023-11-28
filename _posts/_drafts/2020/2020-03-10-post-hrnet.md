---
layout: post
title: "HRnet 图像关键点检测网络结构"
subtitle: 'v1 v2 v2p模型结构解析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-03-10 10:55 
lang: ch 
catalog: true 
categories: documentation
tags:
  - keypoint detection 
  - Time 2020
---

## Introduction
High resolution network(HRnet)主要针对于图像分类、人脸关键点检测、图像分割三个方向进行研究。现有的大多数的方法都是使用`encoder`连接`decoder`的结构，对输入的图像进行特征抽取到一个低分辨率的空间，然后在从低分辨率恢复到高分辨率。这种信息处理的方式对于图像分类、目标检测等相对粗粒度的任务而言可以适用，但对于关键点检测、图像分割等细粒度任务而言，如果能够在特征抽取的同时，保持输入的高分辨率特征，将会得到更好的效果。

## Improvements 
<b>几种经典的多分辨率网络结构设计模式如下：</b> <br>
1 对称结构，先下采样，再上采样，同时使用跳层连接恢复下采样丢失的信息。<br>
2 级联的多层金字塔结构。<br>
3 先使用[strided conv](#strided_conv)下采样，再使用[transform conv](/documentation/2019/10/11/post-transpose-conv/)或者使用[nearest interpolation结合1x1conv](#upsample_conv)的方式对于特征图进行上采样，不使用跳层连接进行数据融合。 <br>
4 [dilated conv](/documentation/2020/03/10/post-dilated-conv/)代替regular conv，相同的参数量增大输入感受野(分辨率)，可以一定程度减少下采样次数，不使用跳层连接进行数据融合。<br>

<center><img src="/img/in-post/hrnet/up_down.pdf" width="100%"></center>

## Base Structure
<b>[HRnet-v1]</b> 
HRnet的结构充分利用了上述现有的网络结构的特点，并且通过在不同分辨率的层次之间构建更加稠密的`shortcut`连接进行连接。

<center><img src="/img/in-post/hrnet/structure.pdf" width="80%"></center>

上图中红色箭头展示的两种连接：`Sequential multi-resolution subnetworks`以及`Parallel multi-resolution subnetworks`是这篇hrnet论文所提出的核心结构创新点。

<center><img src="/img/in-post/hrnet/resolution.pdf" width="80%"></center>

如上图所示：`Repeated multi-scale fusion` hrnet使用了一种跨不同分辨率的连接的方式对于来自不同分辨率层次的特征图进行融合。

<b>[HRnet-v2]</b> 
HRnet-v2的结构overview如下图所示，在v1中介绍的基础上，做了一些改进。

<center><img src="/img/in-post/hrnet/structure_v2.pdf" width="100%"></center>

上图中的部分结构细节如下图所示，其中(c)图给出了一种regular conv结构的等效结构，目的是给(b)中展示的结构做补充说明：

<center><img src="/img/in-post/hrnet/details_v2.pdf" width="80%"></center>

对于图(c)中左边和右边卷积模式的等效性进行解释：

<center><img src="/img/in-post/hrnet/details_v21.pdf" width="100%"></center>

<b>[根据HRnet-v2结构任务需要，对应下面4种输出层结构设计]</b> <br>
注意，对于keypoint detection任务，只有高分辨率的特征图用于产生输出：

<center><img src="/img/in-post/hrnet/details_v22.pdf" width="100%"></center>

## Code
<span id="strided_conv"></span>

### Strided_conv 
```python
def downsample_block(input, planes, scope, has_relu):
    # downsample link in hrnet
    with tf.variable_scope(scope):
        _out = slim.conv2d(input, num_outputs=planes, kernel_size=[3, 3], stride=2,
                           activation_fn=tf.nn.relu if has_relu else None,
                           normalizer_fn=batch_norm, padding='SAME')
    return _out
```
<span id="upsample_conv"></span>

### Upsample_conv 
```python
def upsample_block(input, ratio, planes, scope):
    # upsample link in hrnet
    with tf.variable_scope(scope):
        _out = slim.conv2d(input, num_outputs=planes, kernel_size=[1, 1], stride=1,
                           activation_fn=None, normalizer_fn=batch_norm, padding='SAME')
        shape = _out.shape
        _out = tf.image.resize_nearest_neighbor(_out, (shape[1] * ratio, shape[1] * ratio))
    return _out
```

## Reference
> https://arxiv.org/abs/1902.09212 <br>
> https://arxiv.org/abs/1904.04514 <br>
> https://github.com/yuanyuanli85/tf-hrnet <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
