---
layout: post
title: "auto encoder (machine learning)"
author: "melon"
date: 2019-09-22 10:36
categories: "2019"
tags:
  - machine learning
  - ongoing
---

auto encoder is learning a identity mapping from x to z and then back to x.

the part from x to z can be seen as encoder, which is used to get a suppressed feature
representation of input x.
the part form z back to x' can be seen as decoder to reconstruct x.

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/ae.pdf" width="350"/>

set the activate function as sigmoid, the encoder can be g(x) = sigmoid(wx+b).

the procedure of encoder & decode can not recover full x due to information loss, so the
output x' cannot equal input x. so the auto encoder model can be used to generate new
sample point without supervision.

the target for auto encoder model is to derive a suppressed feature representation of input x,
rather than the x or x'.
the difference between x and x' is used to measure the performance of auto encoder, and middle layer z
is to measure the ability of info suppression regarding input x.

<hr>

### # normalization
there are mainly 2 restrictions methods to normalize the representation of input:  
1 restriction to the hidden layer dimension.  
2 sparsing the params of each node in hidden layer.
  for a given num of hidden layer dimension, could rectify the loss function with:
  $ \sum_{i \in h}KL(\rho | \tilde{\rho_{a}}) $ to achieve the purpose.  
3 add l1/l2 norm term inside loss function, to make the model param sparser or make the param value smaller.

<hr>

### # variants
1 principle component analysis: remove activate function(sigmoid) from g(x), use quadratic loss function,
the auto encoder degenerates to pca.  
2 sparse representation of the input data: the dimension of middle layer z is larger than input x.

<hr>

### # reference
todo: http://127.0.0.1:4000/documentation/2019/09/22/post-auto-encoder/

