---
layout: post
title: "Variational auto encoder - Bayesian view"
subtitle: '原理解析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-09-26 10:09 
lang: ch 
catalog: true 
categories: documentation
tags:
  - unsupervised learning
  - generative model
  - Time 2019
---

## Introduction
[上一篇关于变分编码器的文章](/documentation/2019/09/23/post-variational-auto-encoder/)分两个部分介绍了VAE的损失函数。从假设隐变量$Z$的分布的假设推导出VAE第一部分loss：KL-loss。 又假设输出数据关于隐变量的条件分布，使用最大似然的方式，确定了第二部分loss的形式。思路清晰，但是缺乏整体性。这篇文章旨在从另外的角度出发，优化不同的目标，殊途同归，达到相同的最终目标。　
## Bayesian view
在开始数学推导之前，首先对于将用到的符号进行列举。$x_k$和$z_k$表示随机变量$x$,$z$的第$k$个样本。$x_{(k)}$和$z_{(k)}$表示向量的第$k$个分量。$\mathbb{E}_{x\sim p(x)}[f(x)]$表示对$f(x)$计算期望，其中$x$的分布为$p(x)$。$\mid\mid x\mid\mid ^{2}$表示向量的$l2$范数。$q(x,z)$表示建立在$(x,z)$联合空间上的模型生成联合分布，$p(x,z)$表示$(x,z)$空间上的真实联合分布。

重新明确一下证明的目标：我们有一批数据样本${x_1,x_2,...,x_n}$，整体使用随机变量$X$来描述，我们希望提取“最有价值的特征”$Z$，这样的$Z$能够“最完美”的生成输入随机变量$X$分布里的观测值：

$$
q(x)=\int q(x\mid z)p(z)dz, \quad q(x,z)=q(x\mid z)q(z)
$$

其中，$q(z)$是隐变量先验分布(标准正态)，目的是希望生成分布$q(x)$能够逼近输入真实分布$\tilde{p}(x)$。和上篇vae文章不同的是，**我们这里假设生成$q(x,z)$逼近真实$\tilde{p}(x,z)$**。使用KL散度来算两个分布之间的距离：

$$
KL(p(x,z)\mid\mid q(x,z))=\iint p(x,z) \ln \frac{p(x,z)}{q(x,z)} dzdx
$$

我们希望$q(x,z)$等价于$p(x,z)$，相当于最小化KL散度，将上面公式展开($dx$和$dz$分离)：

$$
KL(p(x,z)\mid\mid q(x,z))=\int \tilde{p}(x)\left[\int p(z\mid x) \ln \frac{\tilde{p}(x) p(z\mid x)}{q(x,z)}dz\right]dx\\
=\mathbb{E}_{x \sim \tilde{p}(x)}\left[\int p(z\mid x) \ln \frac{\tilde{p}(x) p(z\mid x)}{q(x, z)}dz\right]
$$

对上面的公式进行进一步的简化，由于：$\ln \frac{\tilde{p}(x)p(z\mid x)}{q(x,z)}=\ln \tilde{p}(x)+\ln \frac{p(z\mid x)}{q(x, z)}$，带入上式：

$$
=\mathbb{E}_{x\sim \tilde{p}(x)}\left[\int p(z\mid x)\left( \ln \tilde{p}(x)+\ln \frac{p(z\mid x)}{q(x,z)} \right) dz\right]
$$

将两项拆开：

$$
=\mathbb{E}_{x \sim \tilde{p}(x)}\left[\int p(z\mid x) \ln \tilde{p}(x) dz\right]+\mathbb{E}_{x \sim \tilde{p}(x)}\left[\int p(z\mid x) \ln \frac{p(z\mid x)}{q(x,z)} dz\right]
$$

对上式中的第一项继续简化：

$$
=\mathbb{E}_{x \sim \tilde{p}(x)}\left[\ln \tilde{p}(x) \int p(z\mid x) dz\right]=\mathbb{E}_{x\sim \tilde{p}}(x)[\ln \tilde{p}(x)]
$$

分析上式结果：$\tilde{p}(x)$是真实分布，是一个常量，在最小化整体损失函数的过程中可以忽略不计，所以只有第二项起作用。利用$q(x,z)=q(x\mid z)q(z)$对于第二项进行拆分化简得到：

$$
\mathcal{L}=\mathbb{E}_{x \sim \tilde{p}(x)}\left[\int p(z\mid x) \ln \frac{p(z\mid x)}{q(x\mid z) q(z)} dz\right]
$$

再将$ln$分子分母拆开：

$$
=\mathbb{E}_{x \sim \tilde{p}(x)}\left[-\int p(z\mid x) \ln q(x\mid z)dz+\int p(z\mid x) \ln \frac{p(z\mid x)}{q(z)} dz\right]
$$

简写上式：

$$
\mathcal{L}=\mathbb{E}_{x\sim \tilde{p}(x)}\left[\mathbb{E}_{z\sim p(z\mid x)}[-\ln q(x\mid z)]+\mathbb{E}_{z\sim p(z\mid x)}\left[\ln \frac{p(z\mid x)}{q(z)}\right]\right]
$$

$$
=\mathbb{E}_{x \sim \tilde{p}(x)}\left[\mathbb{E}_{z \sim p(z\mid x)}[-\ln q(x\mid z)]+KL(p(z\mid x)\mid\mid q(z))\right]
$$

显然，上面的损失函数即是[上一篇VAE文章](/documentation/2019/09/23/post-variational-auto-encoder/)中分别由两个分布假设得到的loss加和。两种关于VAE构建方式的理解殊途同归。

## Reference
> https://kexue.fm/archives/5343#直面联合分布

> 1 当使用inline数学公式的时候 应当使用`$...$`符号<br>
> 2 当希望数学公式单独成行的时候 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
> 8 编辑超级链接实现文章之间跳转 -> 直接打开另外一篇文章 将localhost:4000后面的地址复制到()中即可<br>