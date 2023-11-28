---
layout: post
title: "Compute receptive field"
subtitle: '感受野计算方式'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-10-03 00:16 
lang: ch 
catalog: true 
categories: documentation
tags:
  - nn toolkit
  - Time 2019
---

## Introduction
感受野是指定的feature map中的**一个像素**关于**输入图像**中的相关区域。padding不影响感受野，stride只影响下一层的感受野。可以通过下面的图片进行直观的理解：

<center><img src="/img/in-post/receptive_field/receptive_field.png" width="60%"></center>

其中，原始图像大小$7\times 7$，经过$$(kernel\_size=3, stride=2)$$的$conv1$以及$$kernal\_size=2$$,$$stride=1$$的$conv2$后，输出特征图大小为$2\times 2$。下面用公式对于感受野进行精确的表达。

## Top-to-down
The receptive field (RF) of layer $n$ is:

$$
RF_n=RF_{n-1}+((k_n-1)*\prod_{i=1}^{n-1}s_i)
$$

where $RF_{n-1}$ is the receptive field of layer $n-1$, $k_n$ is the filter size(height or width, but assuming they are the same here), and $s_i$ is the stride of layer $i$. If want compute the receptive field of layer n, one should compute from the layer 1, then layer 2 ...

## Reference
> https://www.jianshu.com/p/9997c6f5c01e
> https://shawnleezx.github.io/blog/2017/02/11/calculating-receptive-field-of-cnn/

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内