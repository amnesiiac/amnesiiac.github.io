---
layout: post
title: "Variational Auto Encoder"
subtitle: '原理公式推导|简单应用'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-09-23 13:57 
lang: ch 
catalog: true
categories: documentation 
tags:
  - unsupervised learning 
  - generative model
  - Time 2019
---

###  Take a glimpse
VAE的初衷：变分自编码器是一种概率生成模型，它的目标和gan是基本一致的，都是希望构建一个从隐变量$Z$生成和输入数据$X$相关的随机变量($X'$)的模型；通过对于输入数据$X$的分布的编码，实现特征提取的功能，但又不希望将输入的信息学的太死板(overfitting)，这样容易退化成上一篇文章中介绍的AE。<br>
我们希望从输入数据的样本点中拟合出"数据背后真实分布的形态"，并且在它的真实分布中进行采样，生成新数据，然而，这个真实分布是不可解的。

具体地，VAE试图通过encoder部分学习关于输入数据的中的一些特征，encoder的功能可以理解成是特征提取降维，也可以理解成是根据输入的样本由神经网络模型去获得每个输入样本的高斯分布参数(这里强制将输入数据的特征编码为正态分布的组合，类似GMM：任何分布理论上都可以通过高斯混合模型来拟合)，然后将encoder得到的特征分布加入noise，使得学习到的特征分布在一定的范围内波动，这样我们就可以得到用于产生输出的隐变量的分布了，最后通过采样得到隐变量，再经过decoder部分就得到了我们生成的输出。

### Core Obstacle
假设隐变量$Z$服从标准的正态分布，且从隐变量到输出的随机变量$X'$之间的关系为$X'=g(Z)$，那么我们就可以从标准正态分布中经过采样得到若干个$Z1,Z2...,Zn$，于是易知$X1'=g(Z1)$, $X2'=g(Z2)$…$Xn'=g(Zn)$。该怎么进行model training使得我们得到的$X'$的分布和我们所输入的随机变量$X$的分布是一致的(或者说是相关的)呢？

容易想到利用KL散度对于输入$X$输出$X'$的分布差异进行量化，但这行不通，因为输入变量和输出变量都是从各自真实分布中采样得到的样本，我们无法获取它们真实分布表达式，无法算出KL散度。

有两种方式去解决这个问题：既然KL散度的度量方式不行，能否把度量两个分布是否相近似的任务交给神经网络呢(神经网络时代的哲♂学)，gan就是这样做的。那么VAE又是如何实现在模型的学习过程中保证两个随机变量之间距离近似呢？下面通过一些公式+图进行详细量化的解释。

### How To Overcome/Bypass
首先假设我们有一批数据样本${x1,x2,...,xn}$，数据样本背后的真实分布可用$X$来描述。对于一个理想的生成模型来说，我们希望根据已有的数据样本，得到其真实分布$P(X)$，再从$P(X)$中采样即可得到所有可能的$x$。但是这种理想生成数据的路线不能够实现。

因此我们换一种方式去表达所需要的样本空间的真实分布：$P(X)=\sum\_zP(X\mid Z)P(Z)$。其中，$P(X\mid Z)$描述了一个由$Z$(隐变量)来生成$X$(输入变量)的模型。为了简化问题需要，我们假设$Z$服从标准正态分布$N(0,1)$。这样，我们就可以通过从标准正态分布中采样来获得$Z$，再通过生成器$P(X\mid Z)$得到$X$。

**于是目标问题的转化框架如下图所示**：

<img src="/img/in-post/variational_auto_encoder/vae0.jpeg" width="50%">

**现在，将上述简化的问题回归到原始问题**：<br>
隐藏变量$Z$的分布不一定是标准正态分布，对于一个复杂的数据，表征其内在特征的分布不太可能仅仅是一个正态分布那么简单。因此，参考GMM，我们可以做更加泛化一点的假设，即输入样本的真实分布$P(X)$是由很多"专属于每个输入样本$xi$"的正态分布混合得到的。

具体来说，给定一个样本$xk$，假设存在一个专属$xk$的分布$P(Z\mid X\_k)$是一个独立多元正态分布。VAE的论文中也强调了这个想法：

> In this case, we can let the variational approximate posterior be a multivariate Gaussian with a diagonal convariance structure: $\log q_{\phi}\left(z\mid x^{(i)}\right)=\log \mathcal{N}\left(z; \boldsymbol{\mu}^{(i)}, \sigma^{2(i)} \boldsymbol{I}\right)$     
> Ref: "Auto-Encoding Variational Bayes"

现在，每一个输入变量$xk$都有一个正态分布相匹配，我们定义代表输入样本的随机变量$Xk$相应的正态分布形式为：$P(Z\mid X\_k)\sim \mathcal{N}(\mu\_k,\sigma\_k)$。那么怎样确定关于每个$X\_k$及其隐变量的均值和方差呢？没什么思路的情况下，我们就会参考神经网络时代的哲♂学进行解决，不会算的统统交给NN就好。

使用神经网络的模型，建立均值方差表达式如下：$\mu\_k=f\_1(X\_k)$，$log\sigma\_k^2=f\_2(X\_k)$来进行计算，其中$f\_1,f\_2$分别是两个神经网络的inference模型。注意到我们拟合的是$log\sigma\_k^2$而不是直接拟合$\sigma\_k^2$，是因为$\sigma^2>=0$，神经网络拟合它需要加入激活函数对于输出为负的部分进行抑制，而拟合$log\sigma\_k^2$就不需要考虑这个问题。上述的整个过程相应的图示如下(u表示均值，a表示方差)：

<img src="/img/in-post/variational_auto_encoder/vae3.jpeg" width="60%">

对于上面的图中$P(Z\mid X)$向标准正态分布回归的含义是：尽量让$P(Z\mid X)$接近标准正态分布，这样就有$P(Z)=\sum\_X P(Z\mid X)P(X)=\sum\_X \mathcal{N}(0,1)P(X)= \mathcal{N}(0,1)$，我们在开始的时候对latent variable $z$的先验分布所作出的假设在这个步骤中被利用，至于为什么要做这种先验假设呢，我的理解是：**1** 便于我们进行神经网络进行bp优化，如果直接从$\mathcal{N}(\mu,\sigma)$中进行采样，由于$\mu$和$\sigma$作为encoder的输出，就需要对于采样过程进行bp，但是采样过程是不可导的，因此对$z$进行标准化，直接从 $\mathcal{N}(0,1) $中采样，避免了采样过程参与神经网络的bp。**2** 这是一种对于模型加入正则的方式(我称之为先验正则)。

### KL-loss
那么如何让所有的$P(Z\mid X)$都近似为$\mathcal{N}(0,1)$呢？考虑在重构误差的基础之加入下面两个loss，分别使上述分布的均值和方差向标准正态分布靠近。
$$
\mathcal{L}_{\mu}=\mid\mid f_{1} \left(X_{k}\right) \mid\mid ^{2} \mathcal{L}_{\sigma^{2}}=\mid\mid f_{2}\left(X_{k}\right) \mid\mid^{2}
$$

我们的目标是让上面的两个loss尽量接近0，这就等价于使 $P(Z\mid X)\sim \mathcal{N}(0,1)$。 原论文直接计算 $P(Z\mid Xk) \sim  \mathcal{N}(\mu\_k,\sigma\_k^2)$ 和标准正态分布的KL散度。计算结果为：

$$
\mathcal{L}_{\mu, \sigma^{2}}=\frac{1}{2} \sum_{i=1}^{d}\left(\mu_{(i)}^{2}+\sigma_{(i)}^{2}-\log \sigma_{(i)}^{2}-1\right)
$$

对于上式中特定的$i$来说，$KL\left(N\left(\mu, \sigma^{2}\right) \mid\mid N(0,1)\right)$的推导过程如下：

$$
=\int\left( \frac{1}{\sqrt{2 \pi \sigma^{2}}} e^{-(x-\mu)^{2} / 2 \sigma^{2}}\left(\log \frac{e^{-(x-\mu)^{2} / 2 \sigma^{2}} / \sqrt{2 \pi \sigma^{2}}}{e^{-x^{2} / 2} / \sqrt{2 \pi}}\right) \right)dx
$$

$$
=\int\left( \frac{1}{\sqrt{2 \pi \sigma^{2}}} e^{-(x-\mu)^{2} / 2 \sigma^{2}} \log\left( \frac{1}{\sqrt{\sigma^{2}}} \exp \left( \frac{1}{2}\left(x^{2}-(x-\mu)^{2} / \sigma^{2}\right) \right) \right) \right) dx
$$

$$
=\frac{1}{2} \int \left( \frac{1}{\sqrt{2 \pi \sigma^{2}}} e^{-(x-\mu)^{2} / 2 \sigma^{2}}\left(-\log \sigma^{2}+x^{2}-(x-\mu)^{2}/\sigma^2 \right)\right)dx
$$

$$
=-\frac{1}{2} \int\left( \frac{1}{\sqrt{2 \pi \sigma^{2}}} e^{-(x-\mu)^{2} / 2 \sigma^{2}}\cdot \log \sigma^{2}\right)dx +  \frac{1}{2} \int\left( \frac{1}{\sqrt{2 \pi \sigma^{2}}} e^{-(x-\mu)^{2} / 2 \sigma^{2}} \cdot x^2 \right)dx  \\  -\frac{1}{2} \int\left( \frac{1}{\sqrt{2 \pi \sigma^{2}}} e^{-(x-\mu)^{2} / 2 \sigma^{2}} \cdot (x-\mu)^2/\sigma^2 \right)dx
$$

$$
=\frac{1}{2}\left((-\log \sigma^{2})+(\mu^{2}+\sigma^{2})-1\right)
$$

其中第一项是常数，第二项是正态分布的二阶矩，第三项将$\sigma^2$放到分母上，分子是方差的定义所以为1。

现已知$P(Z\mid X\_K)\sim \mathcal{N}(0,1)$，那么下一步需要我们采样得到一个$Z\_k$出来，我们已经知道$P(Z\mid X\_k)$所服从的正态分布的均值和方差，那么$Z\_k$可以从$\mathcal{N}(\mu\_k,\sigma\_k)$分布采样得到，根据下面的公式：

$$
\frac{1}{\sqrt{2 \pi \sigma^{2}}} \exp \left(-\frac{(z-\mu)^{2}}{2 \sigma^{2}}\right)dz = \frac{1}{\sqrt{2 \pi}} \exp \left[-\frac{1}{2}\left(\frac{z-\mu}{\sigma}\right)^{2}\right] d\left(\frac{z-\mu}{\sigma}\right)
$$

说明$(z-\mu)/\sigma=\epsilon$是服从均值为0，方差为1的标准正态分布。因此，从$\mathcal{\mu\_k,\sigma\_k^2}$中采样一个$Z\_k$，相当于从$\mathcal{N}(0,1)$中采样一个$\epsilon$，然后做变换即可$Z\_k=\mu\_k+\epsilon\_k\times \sigma\_k$。

现在已经得到了$Z\_k$，接下来就是decode部分，为了得到$X'$，需要我们知道$P(X\mid Z\_k)$的分布，那么这个分布属于什么分布呢？“Auto-Encoding Variational Bayes”论文给出了两种方案：伯努利分布(01分布)或正态分布(对于NN推导的时候，如果发现想要的变量的分布无法获得，那么就应用NN时代的哲♂学，引入一个computational friendly的分布作为先验即可)。对于二值数据，使用bernouli即可，对于一般数据，使用正态分布。

### Reconstruction-loss
关于decode部分的损失函数的建立，实际是利用最大似然函数的思想，去求解deocder中的参数。对于伯努利分布形式，首先写出伯努利分布及其似然函数如下：

$$
p(\xi)=\left\{\begin{array}{l}{\rho, \xi=1} \\ {1-\rho, \xi=0}\end{array}\right.
$$

$$
q(x\mid z)=\prod_{k=1}^{D}\left(\rho_{(k)}(z)\right)^{x(k)}\left(1-\rho_{(k)}(z)\right)^{1-x(k)}
$$

$$
-\ln q(x\mid z)=\sum_{k=1}^{D}\left[-x_{(k)} \ln \rho_{(k)}(z)-\left(1-x_{(k)}\right) \ln \left(1-\rho_{(k)}(z)\right)\right]
$$

由于$\rho(z)$表示伯努利分布的概率，是一个0-1之间的数，因此想到使用sigmoid函数作为激活函数(将$w\cdot x+b$的输出映射到01之间)，交叉熵作为损失函数即可。对于正态分布模型，同理可以写出其似然函数如下：

$$
q(x\mid z)=\frac{1}{\prod_{k=1}^{D} \sqrt{2 \pi \tilde{\sigma}_{(k)}^{2}(z)}} \exp \left(-\frac{1}{2}\sum_{k=1}^{D} \mid\mid \frac{x-\tilde{\mu\_k}(z)}{\tilde{\sigma\_k}(z)}\mid\mid ^{2}\right)
$$

$$
-\ln q(x\mid z)=\frac{1}{2}\sum_{k=1}^{D}\mid\mid \frac{x-\tilde{\mu}(z)}{\tilde{\sigma}(z)}\mid\mid ^{2}+\frac{D}{2} \ln 2 \pi+\frac{1}{2} \sum_{k=1}^{D} \ln \tilde{\sigma}_{(k)}^{2}(z)
$$

如果进一步加强上述假设，即$P(Z\mid X_k)\sim \mathcal{N}(\mu_k,\sigma_k)$，且固定$\sigma_k$为常数时，并且将上式中的常数项忽略，则可以写成$MSE$的形式如下：

$$
-\ln q(x\mid z)=\frac{1}{2}\sum_{k=1}^{D}\mid\mid \frac{x-\tilde{\mu}(z)}{\tilde{\sigma}(z)}\mid\mid ^{2}
$$

上述伯努利和正态分布损失函数中的$D$在下面python具体实现的过程中就是batch\_size，反向传播的loss需要应用`tf.reduce_mean`函数取平均。


### Summary
**关于`auto-encoder`和`variational-auto-encoder`构建思想**：我们希望能够获得输入(e.g. 一幅图像)的特征，但是特征是抽象的(定性)，因此我们需要构建出一种检验特征好坏的标准(量化)。<br>
**第一种想法**：好的特征能够最大程度的还原原来的图像，这就是`auto-encoder`的建立思想。但是，经过auto-encoder提取的特征可解释性很差，我们不能对于隐藏层(bottleneck)中获得的特征给予描述。<br>
**第二种想法**：利用GMM，任何分布都可以由若干正态分布的混合加以拟合，如果我们在`auto-encoder`中，对于隐藏层加入一个更强的先验，即它们都应该服从$\mathcal{N}(0,1)$分布，而且输入图像所对应的分布能够用隐藏层的混合分布予以表示，那么，提取特征的可解释性就非常好，这就是`variational-auto-encoder`的想法。

**关于隐藏层的服从$N(0,1)$先验的补充说明**：<br>
我们为什么要加入一个比普通$\mathcal{N}(\mu,\sigma)$分布更强的先验，让隐藏层分布接近$\mathcal{N}(0,1)$呢？<br>
**第一点**：如果使用一般的正态分布，让decoder从一般正态分布进行采样，那么显然bp算法需要对encoder部分中计算出$\mu$和$\sigma$同步进行bp参数更新，但是采样过程是不可导的，因此无法进行end-to-end network training；<br>
**第二点**：让隐藏层$Z$的分布从一般正态回归到标准正态的过程，相当于在高斯混合模型的拟合过程中加了一中扰动，相当于一种正则化，能够比单一使用重构loss起到更好的效果。

**关于VAE的两个loss**：KL-loss用来将隐藏层$Z$的分布从一般正态回归到标准正态。reconstruction-loss用消除重构图像和原来图像之间的差异。

**关于VAE的loss推导有另外一种思路**：详见[这篇文章](/documentation/2019/09/26/post-variational-auto-encoder-bayesian-view/)。


### Code
```python
# official variational-auto-encoder 代码详见 ->
# https://github.com/twistfatezz/models/tree/master/research/autoencoder
```
```python
# unofficial implementation of vae
import numpy as np
import tensorflow as tf
import matplotlib.pyplot as plt
import input_data

# %matplotlib inline 

mnist = input_data.read_data_sets('MNIST_data', one_hot=True)
n_samples = mnist.train.num_examples

np.random.seed(0)
tf.set_random_seed(0)

def xavier_init(fan_in, fan_out, constant=1): 
    """ Xavier initialization of network weights"""
    # https://stackoverflow.com/questions/33640581/how-to-do-xavier-initialization-on-tensorflow
    low = -constant*np.sqrt(6.0/(fan_in + fan_out)) 
    high = constant*np.sqrt(6.0/(fan_in + fan_out))
    return tf.random_uniform((fan_in, fan_out), minval=low, maxval=high, dtype=tf.float32)

class VariationalAutoencoder(object):
    """ 
    Variation Autoencoder (VAE) with an sklearn-like interface implemented using TensorFlow.
    This implementation uses probabilistic encoders and decoders using Gaussian 
    distributions and realized by multi-layer perceptrons. The VAE can be learned
    end-to-end. With refers to "Auto-Encoding Variational Bayes".
    """
    def __init__(self, network_architecture, transfer_fct=tf.nn.softplus, 
                 learning_rate=0.001, batch_size=100):
        self.network_architecture = network_architecture
        self.transfer_fct = transfer_fct
        self.learning_rate = learning_rate
        self.batch_size = batch_size
        # build vae
        self.x = tf.placeholder(tf.float32, [None, network_architecture["n_input"]])
        self._create_network()
        # Define loss function based variational upper-bound and corresponding optimizer
        self._create_loss_optimizer()
        # init tf variables 
        init = tf.global_variables_initializer()
        self.sess = tf.InteractiveSession()
        self.sess.run(init)
    
    def _create_network(self):
        # init w&b for encoder 
        network_weights = self._initialize_weights(**self.network_architecture)
        # generate z_mean&log(z_variace^2) through encoder
        self.z_mean, self.z_log_sigma_sq = self._recognition_network(network_weights["weights_recog"], network_weights["biases_recog"])
        # sample z <=> sample epsilon | z~N(u,a)<=>(z-u)/a~N(0,1)
        n_z = self.network_architecture["n_z"]
        eps = tf.random_normal((self.batch_size, n_z), 0, 1, dtype=tf.float32)
        # z=u+a*eps
        self.z = tf.add(self.z_mean, tf.mul(tf.sqrt(tf.exp(self.z_log_sigma_sq)), eps))
        # decoder z->x' 
        self.x_reconstr_mean = self._generator_network(network_weights["weights_gener"], network_weights["biases_gener"])
            
    def _initialize_weights(self, n_hidden_recog_1, n_hidden_recog_2, n_hidden_gener_1, 
                            n_hidden_gener_2, n_input, n_z):
        all_weights = dict()
        all_weights['weights_recog'] = {
            'h1': tf.Variable(xavier_init(n_input, n_hidden_recog_1)),
            'h2': tf.Variable(xavier_init(n_hidden_recog_1, n_hidden_recog_2)),
            'out_mean': tf.Variable(xavier_init(n_hidden_recog_2, n_z)),
            'out_log_sigma': tf.Variable(xavier_init(n_hidden_recog_2, n_z))}
        all_weights['biases_recog'] = {
            'b1': tf.Variable(tf.zeros([n_hidden_recog_1], dtype=tf.float32)),
            'b2': tf.Variable(tf.zeros([n_hidden_recog_2], dtype=tf.float32)),
            'out_mean': tf.Variable(tf.zeros([n_z], dtype=tf.float32)),
            'out_log_sigma': tf.Variable(tf.zeros([n_z], dtype=tf.float32))}
        all_weights['weights_gener'] = {
            'h1': tf.Variable(xavier_init(n_z, n_hidden_gener_1)),
            'h2': tf.Variable(xavier_init(n_hidden_gener_1, n_hidden_gener_2)),
            'out_mean': tf.Variable(xavier_init(n_hidden_gener_2, n_input)),
            'out_log_sigma': tf.Variable(xavier_init(n_hidden_gener_2, n_input))}
        all_weights['biases_gener'] = {
            'b1': tf.Variable(tf.zeros([n_hidden_gener_1], dtype=tf.float32)),
            'b2': tf.Variable(tf.zeros([n_hidden_gener_2], dtype=tf.float32)),
            'out_mean': tf.Variable(tf.zeros([n_input], dtype=tf.float32)),
            'out_log_sigma': tf.Variable(tf.zeros([n_input], dtype=tf.float32))}
        return all_weights
            
    def _recognition_network(self, weights, biases):
        # encoder x->(z_mean,z_variance)
        layer_1 = self.transfer_fct(tf.add(tf.matmul(self.x, weights['h1']), biases['b1'])) 
        layer_2 = self.transfer_fct(tf.add(tf.matmul(layer_1, weights['h2']), biases['b2'])) 
        z_mean = tf.add(tf.matmul(layer_2, weights['out_mean']), biases['out_mean'])
        z_log_sigma_sq = tf.add(tf.matmul(layer_2, weights['out_log_sigma']), biases['out_log_sigma'])
        return (z_mean, z_log_sigma_sq)

    def _generator_network(self, weights, biases):
        # decoder z->x' | p(X|Z)~bernoulli
        layer_1 = self.transfer_fct(tf.add(tf.matmul(self.z, weights['h1']), biases['b1'])) 
        layer_2 = self.transfer_fct(tf.add(tf.matmul(layer_1, weights['h2']), biases['b2'])) 
        x_reconstr_mean = tf.nn.sigmoid(tf.add(tf.matmul(layer_2, weights['out_mean']), biases['out_mean']))
        return x_reconstr_mean
            
    def _create_loss_optimizer(self):
        # reconstruction loss -> maxlikelyhood of P(X=label|Z=input) -> P~bernoulli
        reconstr_loss = -tf.reduce_sum(self.x*tf.log(1e-10+self.x_reconstr_mean)+(1-self.x)*tf.log(1e-10+1-self.x_reconstr_mean), 1)
        # kl-divergence loss -> make the latent z~N(u,a) approch N(0,1)
        latent_loss = -0.5*tf.reduce_sum(1+self.z_log_sigma_sq-tf.square(self.z_mean)-tf.exp(self.z_log_sigma_sq), 1)
        self.cost = tf.reduce_mean(reconstr_loss + latent_loss)
        self.optimizer = tf.train.AdamOptimizer(learning_rate=self.learning_rate).minimize(self.cost)
        
    def partial_fit(self, X):
        opt, cost = self.sess.run((self.optimizer, self.cost), feed_dict={self.x: X})
        return cost
    
    def transform(self, X):
        # 1 encoder test: input -> generate latent data(pca-like transform)
        return self.sess.run(self.z_mean, feed_dict={self.x: X})
    
    def generate(self, z_mu=None):
        # 2 decoder test: generate data by sampling from latent space(we can define the)
        if z_mu is None:
            z_mu = np.random.normal(size=self.network_architecture["n_z"])
        return self.sess.run(self.x_reconstr_mean, feed_dict={self.z: z_mu})
    
    def reconstruct(self, X):
        # 3 encoder&decoder test
        return self.sess.run(self.x_reconstr_mean, feed_dict={self.x: X})


def train(network_architecture, learning_rate=0.001, batch_size=100, training_epochs=10, display_step=5):
    vae = VariationalAutoencoder(network_architecture, learning_rate=learning_rate, batch_size=batch_size)
    # epoch loop
    for epoch in range(training_epochs):
        avg_cost = 0.
        total_batch = int(n_samples / batch_size)
        # per sample loss
        for i in range(total_batch):
            batch_xs, _ = mnist.train.next_batch(batch_size)
            cost = vae.partial_fit(batch_xs)
            avg_cost += cost / n_samples * batch_size
        # loss
        if epoch % display_step == 0:
            print("Epoch:", '%04d' % (epoch+1), "cost=", "{:.9f}".format(avg_cost))
    return vae
```
```python
# test encode-decode models
network_architecture = dict(n_hidden_recog_1=500, n_hidden_recog_2=500, n_hidden_gener_1=500, 
                            n_hidden_gener_2=500, n_input=784, n_z=20)
vae = train(network_architecture, training_epochs=75)
x_sample = mnist.test.next_batch(100)[0]  # input
x_reconstruct = vae.reconstruct(x_sample)  # input->encode->decode->reconstruct
plt.figure(figsize=(8, 12))
for i in range(5):
    plt.subplot(5, 2, 2*i+1)
    plt.imshow(x_sample[i].reshape(28, 28), vmin=0, vmax=1, cmap="gray")
    plt.title("Test input")
    plt.colorbar()
    plt.subplot(5, 2, 2*i+2)
    plt.imshow(x_reconstruct[i].reshape(28, 28), vmin=0, vmax=1, cmap="gray")
    plt.title("Reconstruction")
    plt.colorbar()
plt.tight_layout()
```
```python
# test encoder performance - set latent=2d for visilization
network_architecture = dict(n_hidden_recog_1=500, n_hidden_recog_2=500, n_hidden_gener_1=500, 
                            n_hidden_gener_2=500, n_input=784, n_z=2)
vae_2d = train(network_architecture, training_epochs=75)
x_sample, y_sample = mnist.test.next_batch(5000)
z_mu = vae_2d.transform(x_sample)  # work on encoder
plt.figure(figsize=(8, 6)) 
plt.scatter(z_mu[:, 0], z_mu[:, 1], c=np.argmax(y_sample, 1))
plt.colorbar()
plt.grid()
```
```python
# test decoder performance
nx = ny = 20
x_values = np.linspace(-3, 3, nx)
y_values = np.linspace(-3, 3, ny)
canvas = np.empty((28*ny, 28*nx))
for i, yi in enumerate(x_values):
    for j, xi in enumerate(y_values):
        z_mu = np.array([[xi, yi]]*vae.batch_size)
        x_mean = vae_2d.generate(z_mu)
        canvas[(nx-i-1)*28:(nx-i)*28, j*28:(j+1)*28] = x_mean[0].reshape(28, 28)
plt.figure(figsize=(8, 10))        
Xi, Yi = np.meshgrid(x_values, y_values)
plt.imshow(canvas, origin="upper", cmap="gray")
plt.tight_layout()
```
### Reference
> Article inspiration is from: https://kexue.fm/archives/5253<br>
> Code from: https://jmetzen.github.io/2015-11-27/vae.html<br>
> Thanks this site for useful drawing pics in this article: https://www.processon.com<br>

> 1 当使用inline数学公式的时候 应当使用`$...$`符号<br>
> 2 当希望数学公式单独成行的时候 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
