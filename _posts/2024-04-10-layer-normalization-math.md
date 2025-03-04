---
layout: post
title: "layernorm inference & backpropagation (math, laynorm)"
author: "melon"
date: 2024-04-10 21:05
categories: "2024"
tags:
  - machine learning
  - math
  - llm
  - todo
---

### # layernorm inference

$$
y=\frac{x-\mathrm{E}[x]}{\sqrt{\operatorname{Var}[x]+\epsilon}} * \gamma+\beta
$$

$$
norm=\frac{x-\mathrm{E}[x]}{\sqrt{\operatorname{Var}[x]+\epsilon}}
$$

<hr>

### # layernorm backpropagation
the layernorm inference typically take in following form:

$$
\text{LayerNorm}(x) = w \odot \frac{x - \mu}{\sqrt{\sigma^2 + \epsilon}} + b
$$

where $\odot$ is elementwise multiplication, $\mu$ is the mean, $\sigma^2$ is the variance,
and $\epsilon$ is a small constant to avoid division by zero.

<hr>

### # reference
https://neuralthreads.medium.com/layer-normalization-and-how-to-compute-its-jacobian-for-backpropagation-55a549d5936f  
https://neuralthreads.medium.com/layer-normalization-applied-on-a-neural-network-f6ad51341726  
https://liorsinai.github.io/mathematics/2022/05/18/layernorm.html
