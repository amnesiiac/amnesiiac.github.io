---
layout: post
title: "variational auto encoder (machine learning)"
author: "melon"
date: 2019-09-22 19:36
categories: "2019"
tags:
  - machine learning
---

similar to auto encoder, variational auto encoder also want to retrieve the feature for
a given input, but feature is too abstract to quantify, we need to determine whether a
derived feature is good or bad.

(1) from the view of auto encoder: good feature can best recover original input. however,
the feature derived from auto encoder can hardly be explained.

(2) borrow the view of GMM: every distribution can be solved by a mixture of various norm
distributions. hence, we can set a stronger normalization: the hidden layer conform to
$$\mathcal{N}(0,1)$$ distribution, the input can be represented with mixture of hidden
layer. thus, the derived feature will be more interpretable.

<hr>

### # obstacles to make input/output distribution the same
assuming the latent variable z conform to $$\mathcal{N}(0,1)$$, and the relationship from
latent variable z to output variable $$x'$$ is x'=f(z), we can get latent variable samples
$$z_1,\; z_2...\; z_n$$ from distribution $$\mathcal{N}(0,1)$$, thus we can easily get output
samples using $$x'_1=f(z_1),\; x'_2=f(z_2)...\; x'_n=f(z_n)$$.

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/vae1.pdf" width="300"/>

the question becomes: how to train the model so that the derived distribution of
x' is consistent or related to distribution of input x?

can we use KL divergence for computing the similarity between the input/output distribution?  
the answer is no, cause the input/output are only the sampled data from each reliance
distributions, and we cannot compute KL-loss without the formular expression of the distributions.

another way to solve this is to pass the task of measuring difference between input/output
to neural network, thus we get closer to GAN.

let's findout the solution of this done by VAE in next section.

<hr>

### # ideal solution
given a batch of samples $$x_1,\; x_2...\; x_n$$, the true distribution behind is P(X). for
an ideal generative model, we want to get expression of P(X), then generate data points by
sampling the P(x). however, this ideal way is far from pratical.

<hr>

### # roundabout approach
we choose a roundabout way to get distribution of X: $$P(X)=\sum_zP(X\mid Z)P(Z)$$, in which
$$P(X\mid Z)$$ depict a model to get X using latent variable Z.

assuming $$x_1,\; x_2...\; x_n$$ conform to norm distribution $$\mathcal{N}(\mu,\sigma)$$,
$$z_1,\; z_2...\; z_n$$ conforms to $$\mathcal{N}(0,1)$$, then we could sample from standard
distribution to get z, use trained generative model $$P(X\mid Z)$$ to get x.

the model architecture is as:

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/vae2.pdf" width="560"/>

however, the correspondence between $$x_n’$$ and $$x_n$$ is undetermined,
we only know they should be sampled from the same distribution,
which make it’s hard to compute input/output difference (loss).

<hr>

### # pratical solution
we need stronger normalization rules to solve the problem above:

(1) encoder:  
specifically, for each input feature dimension $$x_k$$, assuming its posterior distribution $$P(Z\mid X_k)$$
as a independant, multivariant norm distribution: $$P(Z\mid X_k)\sim \mathcal{N}(\mu_k,\sigma_k)$$.  
moreover, we assume posterior distribution is conform to standard distribution: $$\mathcal{N}(\mu_k,\sigma_k) \sim \mathcal{N}(0,1)$$,
then the latent variable Z is conform to standard distribution:

$$P(Z)=\sum_X P(Z\mid X)P(X)=\sum_X \mathcal{N}(0,1)P(X)= \mathcal{N}(0,1)$$

as a result, we can directly sample from $$\mathcal{N}(0,1)$$ to get latent variable z.

why posterior distribution should be $$\mathcal{N}(0,1)$$ rather than $$\mathcal{N}(\mu,\sigma)$$?  
if we directly sample from norm distribution, and try to make the VAE as end-to-end
network model, however, the sampling is non-derivable, which make it impossible for end-to-end
network training. thus, using a stronger assumption and separate the model into 2 parts can bypass the problem.

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/vae4.pdf" width="450"/>

(2) decoder:  
after sampling from standard distribution to get latent variable z, we could concatenate
a decoder model $$P(X\mid Z)$$, responsible for getting samples, conform to the input distribution.

<hr>

### # reparameterization trick
sampling from each computed $$\mathcal{N}(\mu,\sigma)$$ to get z is difficult.
using reparameterization trick can alleviate this, given equation as follows:

$$
\frac{1}{\sqrt{2\pi\sigma^2}}\exp\left(-\frac{(z-\mu)^2}{2\sigma^2}\right)dz 
= \frac{1}{\sqrt{2\pi\cdot 1^2}}\exp\left[-\frac{1}{2}\left(\frac{z-\mu}{\sigma}\right)^2\right]d\left(\frac{z-\mu}{\sigma}\right)
$$

use $$(z-\mu)/\sigma=\epsilon$$, then $$\epsilon$$ is conform to $$\mathcal{N}(0,1)$$.  
thus, sampling from $$\mathcal{N}(\mu,\sigma)$$ to get z, is equal to sampling from
$$\mathcal{N}(0,1)$$ to get $$\epsilon$$.

<hr>

### # loss function (todo)
(1) KL-divergence  
(2) reconstruction loss 

<hr>

### # code in action
```text
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
    """
    Xavier initialization of network weights
    https://stackoverflow.com/questions/33640581/how-to-do-xavier-initialization-on-tensorflow
    """
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
    def __init__(self, network_architecture, transfer_fct=tf.nn.softplus, learning_rate=0.001, batch_size=100):
        self.network_architecture = network_architecture
        self.transfer_fct = transfer_fct
        self.learning_rate = learning_rate
        self.batch_size = batch_size
        self.x = tf.placeholder(tf.float32, [None, network_architecture["n_input"]])    # vae inference network
        self._create_network()
        self._create_loss_optimizer()             # define loss function (variational upper-bound and optimizer)
        init = tf.global_variables_initializer()  # init param init values
        self.sess = tf.InteractiveSession()
        self.sess.run(init)
    
    def _create_network(self):
        network_weights = self._initialize_weights(**self.network_architecture)          # init w & b for encoder
        self.z_mean, self.z_log_sigma_sq = self._recognition_network(                    # get z_mean & log(z_variace^2)
            network_weights["weights_recog"], network_weights["biases_recog"]
        )
        n_z = self.network_architecture["n_z"]                                           # sample z: sample from N(0,1)
        eps = tf.random_normal((self.batch_size, n_z), 0, 1, dtype=tf.float32)
        self.z = tf.add(self.z_mean, tf.mul(tf.sqrt(tf.exp(self.z_log_sigma_sq)), eps))  # z=u+a*epsilon
        self.x_reconstr_mean = self._generator_network(                                  # decoder z->x'
            network_weights["weights_gener"], network_weights["biases_gener"]
        )
            
    def _initialize_weights(self, n_hidden_recog_1, n_hidden_recog_2, n_hidden_gener_1, n_hidden_gener_2, n_input, n_z):
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

    # encoder: from x to (z_mean,z_variance)    
    def _recognition_network(self, weights, biases):
        layer_1 = self.transfer_fct(tf.add(tf.matmul(self.x, weights['h1']), biases['b1'])) 
        layer_2 = self.transfer_fct(tf.add(tf.matmul(layer_1, weights['h2']), biases['b2'])) 
        z_mean = tf.add(tf.matmul(layer_2, weights['out_mean']), biases['out_mean'])
        z_log_sigma_sq = tf.add(tf.matmul(layer_2, weights['out_log_sigma']), biases['out_log_sigma'])
        return (z_mean, z_log_sigma_sq)

    # decoder: from z to x' (p(X|Z) ~ bernoulli)
    def _generator_network(self, weights, biases):
        layer_1 = self.transfer_fct(tf.add(tf.matmul(self.z, weights['h1']), biases['b1'])) 
        layer_2 = self.transfer_fct(tf.add(tf.matmul(layer_1, weights['h2']), biases['b2'])) 
        x_reconstr_mean = tf.nn.sigmoid(tf.add(tf.matmul(layer_2, weights['out_mean']), biases['out_mean']))
        return x_reconstr_mean
            
    def _create_loss_optimizer(self):
        reconstr_loss = -tf.reduce_sum(    # reconstruction loss: maxlikelyhood of P(X=label|Z=input) -> P~bernoulli
            self.x*tf.log(1e-10+self.x_reconstr_mean) + (1-self.x)*tf.log(1e-10+1-self.x_reconstr_mean), 1
        )
        latent_loss = -0.5*tf.reduce_sum(  # kl-divergence loss: make N(u,a) approch to N(0,1)
            1+self.z_log_sigma_sq - tf.square(self.z_mean) - tf.exp(self.z_log_sigma_sq), 1
        )
        self.cost = tf.reduce_mean(reconstr_loss + latent_loss)
        self.optimizer = tf.train.AdamOptimizer(learning_rate=self.learning_rate).minimize(self.cost)
        
    def partial_fit(self, X):
        opt, cost = self.sess.run((self.optimizer, self.cost), feed_dict={self.x: X})
        return cost

    # encoder single shot: x -> z
    def transform(self, X):
        return self.sess.run(self.z_mean, feed_dict={self.x: X})

    # decoder single shot: z -> x'
    def generate(self, z_mu=None):
        if z_mu is None:
            z_mu = np.random.normal(size=self.network_architecture["n_z"])
        return self.sess.run(self.x_reconstr_mean, feed_dict={self.z: z_mu})
    
    # encoder + decoder single shot: x -> z -> x'
    def reconstruct(self, X):
        return self.sess.run(self.x_reconstr_mean, feed_dict={self.x: X})
```

training function:
```text
def train(network_architecture, learning_rate=0.001, batch_size=100, training_epochs=10, display_step=5):
    vae = VariationalAutoencoder(network_architecture, learning_rate=learning_rate, batch_size=batch_size)
    for epoch in range(training_epochs):                      # epoch loop
        avg_cost = 0.
        total_batch = int(n_samples / batch_size)
        for i in range(total_batch):                          # per sample loss
            batch_xs, _ = mnist.train.next_batch(batch_size)
            cost = vae.partial_fit(batch_xs)
            avg_cost += cost / n_samples * batch_size
        if epoch % display_step == 0:                         # display loss
            print("Epoch:", '%04d' % (epoch+1), "cost=", "{:.9f}".format(avg_cost))
    return vae
```

test encoder + decoder performance:
```text
network_architecture = dict(n_hidden_recog_1=500, n_hidden_recog_2=500, n_hidden_gener_1=500, 
                            n_hidden_gener_2=500, n_input=784, n_z=20)
vae = train(network_architecture, training_epochs=75)
x_sample = mnist.test.next_batch(100)[0]   # input
x_reconstruct = vae.reconstruct(x_sample)  # input -> encode -> decode -> reconstruct
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

test encoder performance (set latent=2d for visilization):
```text
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

test decoder performance:
```text
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
<hr>

### # reference
http://127.0.0.1:4000/documentation/2019/09/23/post-variational-auto-encoder/  
http://127.0.0.1:4000/documentation/2019/09/26/post-variational-auto-encoder-bayesian-view/
