---
layout: post
title: "Auto-Encoder"
subtitle: '原理简单介绍'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-09-22 13:57 
lang: zh 
catalog: true
categories: documentation
tags:
  - unsupervised learning 
  - generative model
  - Time 2019
---
### 一：Introduction
自动编码器是一种无监督的神经网络模型，它可以通过encode部分学习到输入数据的隐含特征，同时可以通过decode部分将学习到的隐含特征重构原始输入数据。encode部分有数据压缩的性能，这一点从形式上有一点类似于pca主成分分析的降维。另外，由于编解码所产生的一定程度的信息损失，使得ae模型的输入输出不可能完全一致，所以可以用于无监督生成和输入数据不同的新数据。

<center><img src="/img/in-post/auto_encoder/ae.png" width="60%"></center>

如上面的图所示，输入输出神经元的个数相等。输入为$x$，隐藏层为$z$，假设$z$和$x$的关系可以用 $z=g(x)$ 来描述(e.g. $g(x)=sigmoid(w.x+b)$，激活函数也可以是其他类型)，输出 $output$ 和中间隐藏层 $z$ 的关系定义为 $f(.)$ ，那么 $output=f(g(x))=x$ 这个恒等映射就是自动编码器所学要学习的目标。

对于自动编码器，应该关注中间的隐藏层 $z$ 的的分布而不是输入输出。AE学习恒等映射，使用输入输出间分布的差异来衡量AE的性能，而中间层 $z$ 衡量了当自动编解码器的有效信息的压缩能力，或者说$z$的目标是尽量不损失有效信息量的情况下对于原来输入学习出一个最佳的表达。

### 二：AE & its Variants
一般情况下，自编码器的隐藏层的维度是小于输入数据的维度的，因此encode的部分的目的是：尽可能用较小的维度取描述原始数据而尽量不损失数据的信息。特别地，当两层之间变换关系$g(x)$中，将激活函数去掉，而且监督误差为二次型误差时，该网络和主成分分析(pca)等价。另外，如果中间含有多个隐藏层，那么每个隐藏层都是一个受限玻尔兹曼机神经网。

其他的情况下，当隐藏层维度大于输入数据的维度，则可以获得关于输入数据的稀疏编码表示。

上述两种情况，都是通过限制隐藏层的维度来实现对输入表示的约束。第一种方式是直接限制隐藏神经元的个数，另一种则限制隐藏层神经元编码的稀疏性，通过在惩罚函数中增加计算实际激活度$\tilde{\rho}$和期望激活度$\rho$之间的分布差异即$\sum_{i \in h}KL(\rho \| \tilde{\rho_{a}})$来实现。

现在又衍生出很多种方式对于网络进行正则化，如：在惩罚函数中增加L1/L2正则化项，分别可以实现让模型参数更加稀疏和参数的数值更小，以及适当的添加dropout层，以及使用piecewise\_conv和depthwise\_conv代替原来的conv层等。

##### 2.1 自编码器的几种变种形式

|标准(怎样的特征($z$)是好的) | 相应的变种形式|
|  :-: | :-: |
|sparsity(更稀疏的特征) | sparse AE(稀疏自编码器) |
|denoise(噪声更小的特征) | denoising AE(降噪自编码器) |
|regularization(正则约束后的特征) | regularized AE(正则自编码器) |
|repr reg(对抗扰动能力较强的特征) | contractive AE(收缩自编码器)|
|marginalize(边际噪声较小的特征) | marginalized DAE(边际降噪自编码器) |

> See **https://wiseodd.github.io/techblog/2016/12/03/autoencoders/** for details.

__(1) 堆叠自编码器(stacked-auto-encoder)__ <br>
既然自编码器可以学习得到一种关于输入输出近似恒等映射的关系，我们会想到将当前自编码器的输出作为下一个自编码器的输入。假设SAE模型的训练目标是`n->m->k`，那么首先我们训练`n->m->n`，从而得到`n->m`的映射关系，然后再训练`m->k->m`，就得到了`m->k`的映射关系，最终我们就得到了`n->m->k`的模型。<br>
这种layer-wise pre-training + finetuning的方式解决了当时深层网络难以训练的问题。首先进行无监督训练，得到一系列encoder structure作为神经网络的特征提取器，然后通过有监督训练对于分类层(适用于小数据量)或者全部的层(适用于大数据量)参数进行finetune，这种训练方式被认为有利于在有监督阶段加快迭代收敛，并且可以一定程度上解决网络模型过深带来的梯度弥散问题。

<center><img src="/img/in-post/auto_encoder/flow.jpeg" width="40%"></center>

### Code
```python
# todo: 关于auto-encoder及其variants的代码详见 ->
# https://github.com/twistfatezz/models/tree/master/research/autoencoder
```

## Reference
>[1] learning multiple levels of representation and abstraction-hinton, bengio, lecun 2015.
>[2] layer-wise unsuperwised pretraining 逐层非监督预训练
>[3] 知乎问答 为什么稀疏自编码器很少见到多层https://www.zhihu.com/question/41490383

