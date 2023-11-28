---
layout: post
title: "Faster-RCNN"
subtitle: '原理流程解析|region proposal network解析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-10-06 11:54 
lang: ch 
catalog: true 
categories: documentation
tags:
  - object detection
  - Time 2019
---

## Introduction
如果已经读过[RCNN](/documentation/2019/09/29/post-rcnn/)和[fast-RCNN](/documentation/2019/10/06/post-fast-rcnn/)的文章，那么这次将要介绍的faster-RCNN需要我们学习的部分就只集中在region proposal network部分了。faster-RCNN可以看作是fast-RCNN+RPN构成，也就是说，faster-RCNN将fast-RCNN中的region proposal的功能转变成一个专门生成proposal的network来完成。RPN主要包含了三个问题：第一，如何设计RPN，即RPN的结构；第二，如何`摆放`RPN，如何将RPN`塞进`fast-RCNN中，使其构成一个可以end-to-end test的网络；第三，RPN网络以及fast-RCNN的backbone network如何进行训练。

## RPN
使用传统的region proposal方式结合深度神经网络仍然具有空间消耗大，运行时间慢的缺点。为了能够节约资源，将region proposal的部分也交给网络来完成。作者应该是受到之前做region proposal的论文[Scalable, high-quality object detection(Multibox)](https://arxiv.org/pdf/1412.1441.pdf)的启发，并对其进行改进，设计出了RPN。在faster-RCNN论文中作者也将自己的方法和上面论文中的方式做了对比分析。

**首先解决introduction中的第一个问题**：如何设计RPN的结构，能够让它具有selective search的region proposal功能。下图就是原论文中RPN的结构示意图，论文总是惜纸如金，示意图画的`精准而优雅`，但是不适合我这种小白理解，后面我会补充更加详细的RPN结构示意图。

<center><img src="/img/in-post/faster_rcnn/rpn0.png" width="60%"></center>

图中的`conv feature map`就是前一篇文章[fast-RCNN](/documentation/2019/10/06/post-fast-rcnn/)中的`conv5`生成的feature map(假设其大小为mxn)。`sliding window`在论文中被设置为$3\times 3$大小，sliding window相当于一个$3\times 3$的卷积核，对于每个mxn feature map的cell位置都进行一次卷积。`k anchor bbox`是指对于每个mxn feature map中的cell，我们`臆想`其对应输入图像中相应位置的$k$个anchor bbox(其实就是预设几个不同长宽比的框)，论文中$k=9$，那么feature map怎么和输入图像对应起来：见下图，也可以参考[SSD解析](/documentation/2019/10/04/post-ssd/)中的`default box mapping`小节。

<center><img src="/img/in-post/faster_rcnn/anchor.png" width="60%"></center>

最后对于mxn feature map中的每一个cell中的$k$个anchors，产生2个confidence输出，以及4个bbox regression encoding输出，分别用于判断是`前景`还是`背景`，以及进行坐标变换的回归4个参数，faster-RCNN采用了和RCNN同样的坐标回归过程，关于坐标回归的encoder参数理解，详见[RCNN解析](/documentation/2019/09/29/post-rcnn/)。下面是训练RPN的数据流示意图：

<center><img src="/img/in-post/faster_rcnn/train_rpn.png" width="80%"></center>

通过上面的图，以及结合faster-RCNN原论文中的RPN图，可以对于RPN的结构以及训练过程有更好的理解。对于图中的关键点进行解释：$1\times 1$的卷积做了两次，分别用于从上一层feature map中提取相应的confidence信息和bbox regression信息。最后一步的channel为什么是$9\times 2$和$9\times 4$，参见原论文：

> At each sliding-window location, we simultaneously predict $k$ region proposals, so the reg layer has $4k$ outputs encoding the coordinates of $k$ boxes. The cls layer outputs $2k$ scores that estimate probability of object/not-object for each proposal. 

## RPN Loss Function
原论文中讲解的很详细，直接引用原文进行理解，下面引用的原文给出了positive anchor的定义：

> We assign a positive label to two kinds of anchors: (i) the anchor/anchors with the highest Intersection-over-Union(IoU) overlap with a ground-truth box, or (ii) an anchor that has an IoU overlap higher than 0.7 with any ground-truth box.  Anchors that are neither positive nor negative do not contribute to the training objective.

下面根据原文给出negtive anchor的定义：

> We assign a negative label to a non-positive anchor if its IoU ratio is lower than 0.3 for all ground-truth boxes.

RPN训练时候使用的loss function定义如下，之前的[fast-RCNN](/documentation/2019/10/06/post-fast-rcnn/)中已经介绍过这种multi-task loss不再赘述：

$$
L\left(\left\{p_{i}\right\},\left\{t_{i}\right\}\right)=\frac{1}{N_{c l s}} \sum_{i} L_{c l s}\left(p_{i}, p_{i}^{*}\right)+\lambda \frac{1}{N_{r e g}} \sum_{i} p_{i}^{*} L_{r e g}\left(t_{i}, t_{i}^{*}\right)
$$

> The ground-truth label $$p_i^{*}$$ is $1$ if the anchor is positive, and is $0$ if the anchor is negative. $t_i$ is a vector representing the 4 parameterized coordinates of the predicted boundingbox, and $$t_i^{*}$$ is that of the ground-truth box associated with a positive anchor. 

## Pos/Neg Balancing 
关于faster-RCNN中的fast-RCNN部分的训练过程，可以参考[fast-RCNN](/documentation/2019/10/06/post-fast-rcnn/)中的流程图进行理解。我在fast-RCNN论文中没有找到关于mini-batch正负样本均衡的描述，这里在faster-RCNN解析中提一下，直接参考原文就好：

在**训练(固定fast-RCNN进行RPN训练的过程)**过程中，关于如何处理有一部分位于图像外边的anchors：

> During training, we ignore all cross-boundary anchors so they do not contribute to the loss. 在训练的过程中，如果忽略跨越边界的anchors，将会大大地减少总体anchors的数量。如果不排除这些anchors，训练将难以收敛。

在**测试(固定RPN参数进行fast-RCNN部分训练)**过程中，关于如何处理有一部分位于图像外边的anchors：

> During testing, however, we still apply the fully-convolutional RPN to the entire image. This may generate cross-boundary proposal boxes, which we clip to the image boundary. 在测试的过程中，如果proposal的anchors超越了边界，直接沿边界进行crop操作。

> To reduce redundancy, we adopt non-maximum suppression (NMS) on the proposal regions based on their cls scores. We fix the IoU threshold for NMS at 0.7, which leaves us about 2k proposal regions per image.

具体如何**配置用于训练RPN时的mini-batch里面的正负样本个数：**
> It is possible to optimize for the loss functions of all anchors, but this will bias towards negative samples as they are dominate. Instead, we randomly sample 256 anchors in an image to compute the loss function of a mini-batch, where the sampled positive and negative anchors have a ratio of up to 1:1. If there are fewer than 128 positive samples in an image, we pad the mini-batch with negative ones. <br> 
> `explain:` 对于每个输入图像正负bbox共有256个，正负样本比例1:1。和[fast-RCNN](/documentation/2019/10/06/post-fast-rcnn/)一样，选取的正负样本都用于confidence loss的计算，但是只有正样本(相应的bbox)参与bbox regression的计算。

## Model training method
Faster R-CNN的训练，是在已经训练好的model(如VGG_CNN_M_1024，VGG，ZF)的基础上继续进行训练。实际中训练过程分为4个步骤：<br>
第一步：在已经训练好的model上，训练RPN网络。<br>
第二步：利用步骤1中训练好的RPN网络，产生region proposals。<br>
第三步：第一次训练Fast RCNN网络(将selective search替换成了RPN)。<br>
第四步：如果不满足终止条件，转第二步。<br>
训练过程类似于一种“迭代”的过程，faster-RCNN中只循环了2次。循环2次的原因：

> "A similar alternating training can be run for more iterations, but we have observed negligible improvements".

## Code
等待整理...

## Reference
> https://blog.csdn.net/wangpengfei163/article/details/80961275 <br>
> https://zhuanlan.zhihu.com/p/31426458

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内