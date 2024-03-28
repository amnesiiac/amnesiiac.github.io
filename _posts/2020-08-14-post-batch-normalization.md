---
layout: post
title: "batch normalization (machine learning)"
author: "melon"
date: 2020-08-20 21:26
categories: "2020"
tags:
  - machine learning
---

BN is known as batch normalization, is a very important technique for deep learning models
(especially vision related models). It can speed up training, accelerate convergence, and
can be used for anti-overfitting. with BN, the training of network models can use a larger
batch size with larger learning rate.

<hr>

### # why need batch normalization
there's an important assumption in machine learning: IID assumption (indenpendant and identically
distributed assumption).
IID assumes that the training & test data maintain the same distribution:
the empirical & true distribution are the same.

IID can guarantee the model trained by training data can perform well in test data. in other
words, testing model in unpropriate data is waste of time & energy.

batch normalization is used to avoid internal covariance shift between layers of network
model, which can alleviate the low efficiency of training models, and accelerate convergence.

<hr>

### # original idea of BN
(1) understanding the idea of BN by previous research:  
perform whitening operation for the input layer can accelerate
the convergence of nerual networks.
that is, making each dimension of input feature conform to N(0,1) separately will benefit training.
each hidden layer are 'input' of the next, so BN simply apply whitening op to each layer.

(2) understanding the idea of BN by graph:  
given the activation function as sigmoid function, [-infty,x1] or [x2,+infty] are saturated
region in which the gradient updates of parameter is too small to reach convergence, [x1,x2]
are approximately linear region, which is sensitive to gradient updates.

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/bn.pdf" width="360"/>

so the reason for making $$N(\mu, \sigma)$$ to N(0,1) is that: with a probability of 68%
the sample from N(0,1) will get in [-1,1], and a probability of 95% a sample from N(0,1) will
be in [-2,2]\...  
the BN is simply making most of samples of it's input lies in a sensitive state to bp-algorithm
updates.

(3) where should BN structure be placed?  
for relu as activation func, a suitable combination can be conv+BN+relu rather then
conv+relu+BN: relu function is used for half-node suppression, while BN after relu will
break the relu's suppression by shifting zero / negative values back to positive.

for tanh or sigmoid as activation func, a suitable combination can be BN+conv+sigmoid or
BN+conv+tanh: the BN function to drag values back to 0 will weaken the ability for non-linear fitting of
sigmoid/tanh (saturated function).

<hr>

### # BN for training
input of BN using x of a mini-batch: $$\mathcal{B}=\left\{x_1 \ldots x_m \right\}$$,
parameter for learning: $$\gamma,\; \beta$$:

(1) mean of mini-batch:

$$\mu_{\mathcal{B}} \leftarrow \frac{1}{m} \sum_{i=1}^m x_i$$

(2) variance of mini-batch:

$$\sigma_{\mathcal{B}}^2 \leftarrow \frac{1}{m} \sum_{i=1}^m\left(x_i-\mu_{\mathcal{B}}\right)^2$$

(3) normalization of mini-batch by shift & scale (epsilon is added to avoid 0 as divisor):

$$\widehat{x}_i \leftarrow \frac{x_i-\mu_{\mathcal{B}}}{\sqrt{\sigma_{\mathcal{B}}^2+\epsilon}}$$

(4) output of BN by shift & scale:

$$y_i \leftarrow \gamma \widehat{x}_i+\beta \equiv \mathrm{BN}_{\gamma, \beta}\left(x_i\right)$$

when implementing frozen model, the $$\gamma,\; \beta$$ need to be preserved, in case the BN inference
cannot be recovered.

<hr>

### # BN for inference
the BN learnable parameters $$\gamma,\; \beta$$ can be derived by recovering from storation
of each mini-batch of training process. while the batch related param $$\mu_{\mathcal{B}}$$
and $$\sigma_{\mathcal{B}}$$ can not be derived by sampling from test set.

the solution is: based on IID assumption, the test set $$\mu$$ and $$\sigma$$ should be the
same as mean & variance of training set: specifically, store the mean & variance of each
mini-batch during training, then compute math expectation of the stored values to get
inference mean & variance.

(1) compute mean & variance during training:

$$E_B\left[\mu_B\right] \rightarrow E[x], \quad \frac{m}{m-1} E_B\left[\sigma_B^2\right] \rightarrow \operatorname{Var}[x]$$

(2) compute mean & variance during inference:

$$y=\frac{\gamma}{\sqrt{\operatorname{Var}[x]+\epsilon}} \cdot x+\left(\beta-\frac{\gamma \cdot E[x]}{\sqrt{\operatorname{Var}[x]+\epsilon}}\right) \\
=\frac{\gamma \cdot x}{\sqrt{\operatorname{Var}[x]+\epsilon}}-\frac{\gamma \cdot E[x]}{\sqrt{\operatorname{Var}[x]+\epsilon}}+\beta \\
=\frac{\gamma \cdot(x-E[x])}{\sqrt{\operatorname{Var}[x]+\epsilon}}+\beta=\gamma \cdot \frac{x-E[x]}{\sqrt{\operatorname{Var}[x]+\epsilon}}+\beta$$

(3) the inference format above is same as training:

$$y^{(k)}=\gamma^{(k)} \hat{x}^{(k)}+\beta^{(k)}$$

<hr>

### # code in action
(1) api:tf.nn.moments() & tf.nn.batchnormalization() to implement BN:
```text
import tensorflow as tf

def bn_layer(x, is_training, name='BatchNorm', moving_decay=0.9, eps=1e-5):
    # get input dimension
    shape = x.shape
    assert len(shape) in [2, 4]  # 2: fc, 4: conv
    param_shape = shape[-1]

    # init gamma & beta
    gamma = tf.get_variable('gamma', param_shape, initializer=tf.constant_initializer(1))
    beta  = tf.get_variable('beat', param_shape, initializer=tf.constant_initializer(0))

    # compute mean & var of mini-batch
    axes = list(range(len(shape)-1))
    batch_mean, batch_var = tf.nn.moments(x, axes, name='moments')

    # using moving avg of training to compute mean & var
    ema = tf.train.ExponentialMovingAverage(moving_decay)

    def mean_var_with_update():
        ema_apply_op = ema.apply([batch_mean, batch_var])
        with tf.control_dependencies([ema_apply_op]):
            return tf.identity(batch_mean), tf.identity(batch_var)

    # train: update mean & var, test: using mean & var computed from train set
    mean, var = tf.cond(tf.equal(is_training, True), mean_var_with_update,
                        lambda: (ema.average(batch_mean), ema.average(batch_var)))
    return tf.nn.batch_normalization(x, mean, var, beta, gamma, eps)
```


(2) tf.contrib.layers.batch_norm() to implement BN:
```text
import tensorflow as tf

def batch_norm(x, epsilon=1e-5, momentum=0.9, train=True, name="batch_norm"):
    with tf.variable_scope(name):
        epsilon=epsilon, momentum=momentum, name=name
    return tf.contrib.layers.batch_norm(x,
                                        decay=momentum,              # moving avg method
                                        updates_collections=None,    # tf.GraphKeys.UPDATE_OPS or None
                                        epsilon=epsilon,             # 1e-5
                                        scale=True,                  # enable gamma
                                        is_training=train,           # train or test (compute or restore)
                                        scope=name)
```

<hr>

### # applications
large language models due to large input & output, has no problem of covariance shift.

<hr>

### # reference
batch normalization paper: https://arxiv.org/abs/1502.03167
