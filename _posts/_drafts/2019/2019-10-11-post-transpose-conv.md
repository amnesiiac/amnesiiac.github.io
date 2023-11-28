---
layout: post
title: "Transpose convolution"
subtitle: '原理解析|图解'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-10-11 10:52 
lang: ch 
catalog: true 
categories: documentation
tags:
  - conv 
  - Time 2019
---

## Introduction
在`dense prediction`的过程中，需要我们将输入图像进行特征提取，然后再进行上采样得到和输入图像分辨率一致的输出，用于完成`segmentation`，`keypoint detection`等任务。通常上采样有两种方式，一种是直接使用插值的方式，如双线性插值，3次样条插值等；另一种方式，神经网络中，被广泛的采用，就是本文的主角：`transpose convolution`，又称作反卷积。

「**普通卷积的计算方式及其矩阵形式**」下面的图展示了通常理解的卷积运算的计算表示：

<center><img src="/img/in-post/deconv/conv.pdf" width="90%"></center>

对于计算机而言，如果采用上面的计算方式，会大大降低卷积运算的效率。为了优化上面的计算方式，可以将卷积运算转化成矩阵运算的形式，这样可以最大程度利用GPU的特性，实现计算的加速。如下图：

<center><img src="/img/in-post/deconv/matrix_form_conv.pdf" width="90%"></center>

「**转置卷积的计算方式及其矩阵形式**」下面的图展示了通常理解的转置卷积(反卷积)运算的计算表示：

<center><img src="/img/in-post/deconv/deconv.pdf" width="90%"></center>

对于计算机而言，如果采用上面的计算方式，会大大降低卷积运算的效率。为了优化上面的计算方式，可以将卷积运算转化成矩阵运算的形式，这样可以最大程度利用GPU的特性，实现计算的加速。如下图：

<center><img src="/img/in-post/deconv/matrix_form_deconv.pdf" width="90%"></center>

我们可以直观的看到，output的feature map要比原来的input feature map要大，这样就实现了类似上采样的操作。本文中介绍的是最简单的一种转置卷积的方式，如果设定了kernel的padding以及stride之后，会产生不同的output结果。详见[这个repo](https://github.com/twistfatezz/conv_arithmetic)。 

另外补充说明，关于卷积的矩阵表达形式不唯一，可以写成$$y=w*x$$的形式，也可以写成$$y=x*w$$的形式，我推荐写成后面的方式，即本文章介绍的矩阵运算的方式，因为这种形式比较容易推导反向传播的公式。如果想要了解为什么反卷积的一种更加`科学`的叫法是`转置卷积`，请移步[这个知乎专栏](https://zhuanlan.zhihu.com/p/34453588)中，以$y=w*x$的方式对于转置卷积的介绍。

## Some Details
1 if `tf.nn.conv2d_transpose` stride=1, padding zeros around the input feature map. if stride>1, padding zeros into the gap between each cell. 这个fork过来的[repo](https://github.com/twistfatezz/conv_arithmetic)有生动形象的动图。

## Code
tensorflow中已经为我们封装好了转置卷积的函数：
```python
tf.nn.conv2d_transpose(value, filter, output_shape, strides, padding="SAME", data_format="NHWC", name=None)
# 1 首先根据filter stride padding计算output_shape 然后再使用此函数 - 如果output_shape的形状不对 则报错 
# 2 output shape计算检查 将output feature map作为输入 stride作为卷积的stride 然后进行和kernel的卷积操作 得到的input大小应该和实际相同
# 3 为什么需要指定output_shape参数? -> 固定了filter strides padding之后, 从feature map的反卷积的结果不唯一 即有多种feature map进行卷积能够得到同样的feature map.
# 4 题外话: 关于padding='SAME'和padding='VALID' padding='SAME' 
# 如果kernel按照步长移动到最后 feature map元素不够最后一个kernel的conv运算 -> same的方式是优先在左上补0(尽量保证四周均匀) -> valid方式是舍弃右下的元素
```

## Reference
> Kumar Chellapilla, Sidd Puri, Patrice Simard. High Performance Convolutional Neural Networks for Document Processing. Tenth International Workshop on Frontiers in Handwriting Recognition, Université de Rennes 1, Oct 2006, La Baule (France).inria-00112631.

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内