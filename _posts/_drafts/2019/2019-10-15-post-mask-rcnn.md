---
layout: post
title: "Mask-RCNN"
subtitle: '原理解析|结合代码分析结构'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-10-16 9:53 
lang: ch 
catalog: true 
categories: documentation
tags:
  - object detection 
  - Time 2019
---

## Introduction
Mask-rcnn是一个主要用于`instance segmentation`任务的算法，同时兼顾`object detection`的功能。实际使用的过程中，发现效果非常好，是一个里程碑式的经典框架。它结合了多个论文中的精髓思想，融合而成，需要我们用庖丁解牛的方式来分析，将大化小，逐个击破。本文将结合github中star最多的代码([matterport](https://github.com/matterport/Mask_RCNN))来分块进行分析。对于mask-rcnn中借鉴了其他论文的部分，采用给出博客中相应链接的形式，做较少的描述。对于mask-rcnn中初创观点、思想方法，做详尽描述。mask-rcnn是一个`高达模型`，每个组件都是能够独立构成其他论文的核心，通过科学的组装，能够真正达到state-of-the-art的效果。

## Structure
在分部分详解mask-rcnn之前，需要我们对于其整体的框架进行宏观上的认识。如下图所示：

<center><img src="/img/in-post/mask_rcnn/structure.pdf" width="100%"></center>

## RPN
Region proposal network(RPN)是[faster-rcnn](/documentation/2019/10/06/post-faster-rcnn/)中提出的用来代替[selective search](/documentation/2019/10/01/post-region-proposal/)的结构。由由于faster-rcnn已经介绍了rpn的结构，对于其结构，这篇文章不做赘述，这篇文章主要对于FPN+RPN的结构搭配做整合分析，并且分为train/test的两个阶段进行介绍(有些结构在train/test的数据流有一些差异)。
### train_rpn
训练rpn的步骤和[faster-rcnn]类似，不再赘述。
### test_rpn(train_whole_network)
因为这些roi可能是有各个特征层产生的Anchor，所以，现在需要将这些roi映射回FPN产生的特征图(pyramid feature map)上，即需要根据rpn结构的输出，判断，这个roi应该属于哪一层pyramid feature map产生的。简而言之，是根据下面的公式产生的。

$$
k=\left[k_0+log_2(\sqrt {wh}/244)\right ]
$$

其中$w,h$分别表示roi宽度和高度，$k$是这个roi应属于的第$k$个特征层，$k_0$是$w,h=224,224$时映射的对应level的feature map，一般取为4，即对应着P4，至于为什么使用224，一般解释为是因为这是ImageNet的标准图片大小，比如现在有一个roi是$112\times 112$，则利用公式可以计算得到$k=3$，即这个roi对应回到P3层feature map。现在得到了此roi对应的feature map，就可以根据据此提供相应的region of interests以进行下一步的roi-align操作。

## ROI-Align
roi-align是mask-rcnn在模型构建技巧中的创新点。它的目的基本和roi-pooling一致，将不同尺度输入的feature map经过max pooling或avg pooling处理成相同尺度的输出，方便使用full-connected层进行classification & bbox regression。作者在论文中强调，对于基本的classification & bbox regression这种`粗粒度`任务，roi-pooling就可以得到很好的效果，但是对于mask prediction任务如semantic segmentation或者instance segmentation等`pixel-wise`任务而言，引入的误差太大，不能忽略。

「**roi pooling approximation**」给定一张输入图像($800\times 800$)，图像中的检测目标的ground truth bbox为$665\times 665$，经过vgg-16的5个pooling得到的feature map尺度为$(800/32,800/32)=(25,25)$。将bbox对应到feature map上得到相应的尺度为$(665/32,665/32)=(20.78,20.78)$，通过`取整`为$(20,20)$。我们需要通过roi-pooling将feature map中的bbox映射成$7\times 7$的feature vector，于是，每个feature vector中的小格子大小为$(20/7,20/7)=(2.86,2.86)$，`取整`为$(2,2)$。两个取整的过程中引入两次量化误差。

「**roi align methods**」
roi align为了能够保留原feature map中的bbox的映射关系，将roi-pooling中进行的取整操作去除。运算过程如下：给定输入图像($800\times 800$)，图像目标的gt_bbox为$665\times 665$，feature map尺寸为$(800/32,800/32)=(25,25)$。将bbox对应到feature map上得到相应尺度为$(665/32,665/32)$，在通过roi-pooling将bbox映射成$7\times 7$的feature vecotr，于是feature vector中的小格子大小为$(665/32/7,665/32/7)=(2.96875,2.96875)$。那么得到的浮点数位置相应的数值该怎么计算呢?下面的图像展示了将bbox经过roi-align映射成一个2x2的feature vector的过程，使用$a$,$b$,$c$,$d$四个点并利用[bilinear_interpolation](/documentation/2019/10/15/post-interpolation-algorithm/)的方法推断$p$点的数值：

<center><img src="/img/in-post/mask_rcnn/roi_align.png" width="80%"></center>

「**NMS**」关于nms算法及其变种，见这个总结[NMS](/documentation/2019/10/01/post-non-maximum-suppression/)。

后续等待整理...

## Reference

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内