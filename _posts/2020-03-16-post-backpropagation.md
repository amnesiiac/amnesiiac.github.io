---
layout: post
title: "back-propagation algorithm (neural networks, math)"
author: "melon"
date: 2020-03-16 22:26
categories: "2020"
tags:
  - neural network
  - math
---

this article mainly covers the basic neural network back-propagation algorithm.

<hr>

### # math walk for the bp algorithm pipeline
1 given a dataset contains $m$ samples:
$ \{(x^{(1)},y^{(1)}),(x^{(2)},y^{(2)}),\cdots, (x^{(m)},y^{(m)})\} $,
in which x represents the sample data, y represents the label.

<p style="margin-bottom: 20px;"></p>

2 setup the loss function as the squared error cost function:

$$
J(W,b;x,y)=\frac{1}{2}||h_{W,b}(x)-y||^2  \tag{1}
$$

in which $h_{W,b}(x)$ is the neural network output, y is the label of dataset.

<p style="margin-bottom: 20px;"></p>

4 setup the loss function for a batch of samples (m), with weights normalization alongside:

$$
J(W,b)=\frac{1}{m}\sum_{i=1}^mJ(W,b;x^{(i)},y^{(i)})+\frac{\lambda}{2}\sum_{l=1}^{n_{l-1}}\sum_{i=1}^{s_l}\sum_{j=1}^{s_{l+1}}(W_{ij}^{(l)})^2 \tag{2} \\
=\frac{1}{m}\sum_{i=1}^m(\frac{1}{2}||h_{W,b}(x^{(i)})-y^{(i)}||^2)+\frac{\lambda}{2}\sum_{l=1}^{n_{l-1}}\sum_{i=1}^{s_l}\sum_{j=1}^{s_{l+1}}(W_{ij}^{(l)})^2
$$

in which: i denotes the i-th sample; l denotes the layer l of total $n_l$ network layers;
$s_l$ denotes the number of node in the layer l; $W_{ij}^l$ denotes the weight from the
i-th node of layer l to the j-th node of the layer l+1, $W_{kj}^l$ denotes the weight
from the k-th node of layer l to the j-th node of the layer l+1.

<p style="margin-bottom: 20px;"></p>

5 setup update equation for weights & biases separately:  

$$
W_{i,j}^{(l)}=W_{i,j}^{(l)}-\alpha \frac{\partial}{\partial W_{i,j}^{(l)}}J(W,b), \quad  b_i^{(l)}=b_i^{(l)}-\alpha \frac{\partial}{\partial b_i^{(l)}}J(W,b) \tag{3}
$$

in which $\alpha$ denotes the learning rate.

<p style="margin-bottom: 20px;"></p>

6 solve the partial derivative of loss function towards w & b by bringing equation 2 to 3:

$$
\frac{\partial}{\partial W_{i,j}^{(l)}}J(W,b)=\frac{1}{m}\sum_{i=1}^{m}\frac{\partial}{\partial W_{i,j}^{(l)}}J(W,b;x^{(i)},y^{(i)})+\lambda W_{i,j}^{(l)} \tag{4} \\
\frac{\partial}{\partial b_i^{(l)}}J(W,b)=\frac{1}{m}\sum_{i=1}^m\frac{\partial}{\partial b_i^{(l)}}J(W,b;x^{(i)},y^{(i)})
$$

<p style="margin-bottom: 20px;"></p>

7 for simplicity, $h_{W,b}(x)$ is written as $a_i^{n_l}$ to denote the activation of the
last layer in network model, let $z_i^{n_l}$ denote the variable of layer $n_l$ to be activated,
then the following equation established:

$$
a_i^{n_l}=f(z_i^{n_l})=h_{W,b}(x) \tag{6}
$$

in which $f$ is the activation funtion.

<p style="margin-bottom: 20px;"></p>

8 for simplicity, the below concluded formula is targeted to solve the bp correction
for gradient from single sample $(x,y)$ loss rather than the batch loss.
the general form of the back-propagated correction for the i-th node in layer $n_l$ is as:

$$
\delta_i^{n_l}=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}||y-h_{W,b}(x)||^2=-(y_i-a_i^{n_l})\cdot f'(z_i^{n_l}) \tag{5}
$$

the steps to work the above equation out:  
8.1 set batch size as 1 from eq-2, the partial derivative for loss to i-th node
in layer $n_l$ is as:

$$
\delta_i^{n_l}=\frac{\partial}{\partial z_i^{n_l}}J(W,b;x,y)=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}||y-h_{W,b}(x)||^2 \tag{6} \\
=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}\sum_{j=1}^{S_{n_l}}(y_i-a_j^{(n_l)})^2=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}\sum_{j=1}^{S_{n_l}}(y_i-f(z_j^{(n_l)}))^2
$$

8.2 it's easy to find out: only when j=i, the partial derivative towards $z_i^{(n_l)}$
result in non-zero value, all rest condition result in 0.
thus, the above equation can be simplified as:

$$
\delta_i^{n_l}=\frac{\partial}{\partial z_i^{n_l}}J(W,b;x,y)=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}||y-h_{W,b}(x)||^2 \tag{7} \\
= -(y_i-f(z_i^{(n_l)}))\cdot f'(z_i^{(n_l)})=-(y_i-a_i^{(n_l)})\cdot f'(z_i^{(n_l)})
$$

hence the conclusion formula eq-5 is derived.

<p style="margin-bottom: 20px;"></p>

9 according to the law of chain derivation, the delta towards the i-th node of layer l is as:

$$
\delta_i^{(l)}=(\sum_{j=1}^{s_{l+1}}W_{ij}^{(l)}\delta_j^{(l+1)})f'(z_i^{(l)}) \tag{8}
$$

the steps to work the above equation out:  
firstly, try derive the delta of the sample towards i-th node of layer $n_{l-1}$:

$$
\delta_i^{(n_{l-1})}=\frac{\partial}{\partial z_i^{(n_{l-1})}}J(W,b;x,y)=\frac{\partial}{\partial z_i^{(n_{l-1})}}\frac{1}{2}||y-h_{W,b}(x)||^2 \tag{9} 
$$

given eq-6 and the last layer activation value is the model output: $a_j^{n_l}=h_{W,b}(x)$, we have:

$$
=\frac{\partial}{\partial z_i^{(n_{l-1})}}\frac{1}{2}\sum_{j=1}^{S_{n_l}}(y_i-a_j^{n_l})^2=\frac{1}{2}\sum_{j=1}^{S_{n_l}}\frac{\partial}{\partial z_i^{(n_{l-1})}}(y_j-f(z_j^{n_l}))^2 \tag{10}
$$

according to the law of chain derivation:

$$
=-\sum_{j=1}^{S_{n_l}}(y_j-f(z_j^{(n_l)}))\cdot \frac{\partial}{\partial z_i^{(n_{l-1})}}f(z_j^{n_l})=-\sum_{j=1}^{S_{n_l}}(y_j-f(z_j^{(n_l)}))\cdot f'(z_j^{(n_l)})\cdot \frac{\partial z_j^{(n_l)}}{\partial z_i^{(n_{l-1})}} \tag{11}
$$

given eq-5 and network model inference formula $z_j^{n_l}=\sum_{k=1}^{S_{n_{l-1}}}f(z_k^{n_{l-1}})\cdot W_{kj}^{n_{l-1}}$:

$$
=-\sum_{j=1}^{S_{n_l}}(y_j-a_j^{n_l})) \cdot f'(z_j^{(n_l)}) \cdot \frac{\partial z_j^{(n_l)}}{\partial z_i^{(n_{l-1})}} = \sum_{j=1}^{S_{n_l}}\delta_j^{(n_l)}\cdot \frac{\partial z_j^{(n_l)}}{\partial z_i^{n_{l-1}}} \tag{12} \\
=\sum_{j=1}^{S_{n_l}}\delta_j^{n_l} \cdot \frac{\partial}{\partial z_i^{(n_{l-1})}}(\sum_{k=1}^{S_{n_{l-1}}}f(z_k^{n_{l-1}}) \cdot W_{kj}^{n_{l-1}})
$$

only when k=i, the value of $\frac{\partial f(z_k^{(n_{l-1})})}{\partial z_i^{(n_{l-1})}}$ is non-zero, suppress k in above eq:

$$
=\sum_{j=1}^{S_{n_l}}\delta_j^{n_l}\cdot W_{ij}^{n_{l-1}}\cdot f'(z_i^{n_{l-1}})=(\sum_{j=1}^{S_{n_l}}W_{ij}^{n_{l-1}}\delta_j^{(n_l)})f'(z_i^{n_{l-1}}) \tag{13}
$$

re-organize the symbols in above formula, replace symbols $n_{l-1}$ & $n_l$ with l & l+1:

$$
\delta_i^{(l)}=(\sum_{j=1}^{s_{l+1}}W_{ij}^{(l)}\delta_j^{(l+1)})f'(z_i^{(l)}) \tag{14}
$$

the eq-8 get proved, and we get an iterable formula to derive the delta
of i-th node in layer l by the already computed & cached delta for all nodes in layer l+1
that connected with it.

<p style="margin-bottom: 20px;"></p>

10 finally, we could derive the partial derivation towards w & b of layer l by
the iterable formula:  
10.1 partial derivation of single sample x,y towards $W_{ij}^{(l)}$:

$$
\frac{\partial}{\partial W_{ij}^{(l)}}J(W,b;x,y)=a_j^{(l)}\delta_i^{(l+1)}  \tag{15}
$$

$$
proof:   \quad \frac{\partial}{\partial z_i^{(l)}}J(W,b;x,y)=\delta_i^{(l)} \quad w.r.t\;eq.6 \\
thus,    \quad \frac{\partial}{\partial W_{ij}^{(l)}}J(W,b;x,y)=\frac{\partial J(W,b;x,y)}{\partial z_i^{(l)}} \cdot \frac{\partial z_i^{(l)}}{\partial W_{ij}^{(l)}} \\
moreover,\quad \frac{\partial z_i^{(l)}}{\partial W_{ij}^{(l)}}=a_j^{(l)}
$$

10.2 partial derivation of single sample x,y towards $b_{i}^{(l)}$:

$$
\frac{\partial}{\partial b_{i}^{(l)}}J(W,b;x,y)=\delta_i^{(l+1)}  \tag{16}
$$

$$
proof:   \quad \frac{\partial}{\partial z_i^{(l)}}J(W,b;x,y)=\delta_i^{(l)} \quad w.r.t\;eq.6 \\
thus,    \quad \frac{\partial}{\partial b_{i}^{(l)}}J(W,b;x,y)=\frac{\partial J(W,b;x,y)}{\partial z_i^{(l)}} \cdot \frac{\partial z_i^{(l)}}{\partial b_{i}^{(l)}} \\
moreover,\quad \frac{\partial z_i^{(l)}}{\partial b_{i}^{(l)}}=1
$$

to this point, the partial derivative of single sample loss of towards weight & bias in
any layer of the network model. with the delta derived, we can resort to eq-3 for updates
of the model parameters.

<hr>

### # reference
ufldl tutorial
