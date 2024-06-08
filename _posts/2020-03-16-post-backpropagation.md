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

this article mainly summarize the neural network basic algorithm back-propagation.

<hr>

### # preparations
1 samples: given a dataset contains $m$ \\( \alpha \\) samples:
$ \{(x^{(1)},y^{(1)}),(x^{(2)},y^{(2)}),\cdots, (x^{(m)},y^{(m)})\} $,
in which x represents the sample data, y represents the label.

<hr>

### # algorithm details
1 setup the loss function for a single data sample:  
$$h_{W,b}(x)$$ is the neural network output, y is the label of dataset.
assuming the loss function format as the squared err cost function:

$$
J(W,b;x,y)=\frac{1}{2}||h_{W,b}(x)-y||^2
$$

<p style="margin-bottom: 20px;"></p>

2 setup the loss function for given m samples:  
based on the above formula, setup the loss function with normalization alongside:

$$
J(W,b)=\frac{1}{m}\sum_{i=1}^mJ(W,b;x^{(i)},y^{(i)})+\frac{\lambda}{2}\sum_{l=1}^{n_l-1}\sum_{i=1}^{s_l}\sum_{j=1}^{s_{l+1}}(W_{ij}^{(l)})^2 \\
=\frac{1}{m}\sum_{i=1}^m(\frac{1}{2}||h_{W,b}(x^{(i)})-y^{(i)}||^2)+\frac{\lambda}{2}\sum_{l=1}^{n_l-1}\sum_{i=1}^{s_l}\sum_{j=1}^{s_{l+1}}(W_{ij}^{(l)})^2
$$

in which: i represents the i-th sample; l represents the l-th layer of total $$n_l$$ neural
network layers; $$s_l$$ represents the number of node in the l-th layer;
$$W_{ij}^l$$ represents the weight from the i-th node of l-th layer to
the j-th node of the l+1 layer,
$$W_{kj}^l$$ represents the weight from the k-th node of l-th layer to
the j-th node of the l+1 layer.

<p style="margin-bottom: 20px;"></p>

3 setup neural network update formular for weights & biases:  

$$
W_{i,j}^{(l)}=W_{i,j}^{(l)}-\alpha \frac{\partial}{\partial W_{i,j}^{(l)}}J(W,b), \quad  b_i^{(l)}=b_i^{(l)}-\alpha \frac{\partial}{\partial b_i^{(l)}}J(W,b)    \tag{3}
$$

in which $$\alpha$$ represents learning rate.

<p style="margin-bottom: 20px;"></p>

4 resolve the partial derivative of the loss function regarding weights & biases  

$$
\frac{\partial}{\partial W_{i,j}^{(l)}}J(W,b)=\frac{1}{m}\sum_{i=1}^{m}\frac{\partial}{\partial W_{i,j}^{(l)}}J(W,b;x^{(i)},y^{(i)})+\lambda W_{i,j}^{(l)} \\
\frac{\partial}{\partial b_i^{(l)}}J(W,b)=\frac{1}{m}\sum_{i=1}^m\frac{\partial}{\partial b_i^{(l)}}J(W,b;x^{(i)},y^{(i)})
$$

5 take single sample $$(x,y)$$ regarding the w & b, solve the generic format of
the residual error for the $$n_l$$ layer:  

下面公式中，为了方便书写，$h_{W,b}(x)$简记为$a_i^{n_l}$，表示最后一层的激活值，
令$z_i^{n_l}$表示最后一层的待激活值，则有如下关系成立：
$a_i^{n_l}=f(z_i^{n_l})=h_{W,b}(x)$成立，其中$f$表示激活函数。

the conclusion:

$$
\delta_i^{n_l}=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}||y-h_{W,b}(x)||^2=-(y_i-a_i^{n_l})\cdot f'(z_i^{n_l})    \tag{5}
$$

the principle of the above conclusion:

$$
\delta_i^{n_l}=\frac{\partial}{\partial z_i^{n_l}}J(W,b;x,y)=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}||y-h_{W,b}(x)||^2 \\
=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}\sum_{j=1}^{S_{n_l}}(y_i-a_j^{(n_l)})^2=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}\sum_{j=1}^{S_{n_l}}(y_i-f(z_j^{(n_l)}))^2
$$

上面的式子只有当j=i的时候对$$z_i^{(n_l)}$$求解偏导数才有数值否则为0, 所以上面的式子等价于

$$
\delta_i^{n_l}=\frac{\partial}{\partial z_i^{n_l}}J(W,b;x,y)=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}||y-h_{W,b}(x)||^2  \tag{6}
$$

$$
...
$$

$$
-(y_i-f(z_i^{(n_l)}))\cdot f'(z_i^{(n_l)})=-(y_i-a_i^{(n_l)})\cdot f'(z_i^{(n_l)})
$$

hence the conclusion formula is derived.

6 根据链式求导法则 求解神经网络第$l$层第$i$个节点神经元上的bp残差的通式**

$$
结论\quad \delta_i^{(l)}=(\sum_{j=1}^{s_{l+1}}W_{ij}^{(l)}\delta_j^{(l+1)})f'(z_i^{(l)})    \tag{6}
$$

$$
公式(6)的推导:\delta_i^{(n_l-1)}=\frac{\partial}{\partial z_i^{(n_l-1)}}J(W,b;x,y)=\frac{\partial}{\partial z_i^{(n_l-1)}}\frac{1}{2}||y-h_{W,b}(x)||^2\\ 根据公式(1)和最后一层的激活值=神经网络的输出值:a_j^{n_l}=h_{W,b}(x)即可得到\\=\frac{\partial}{\partial z_i^{(n_l-1)}}\frac{1}{2}\sum_{j=1}^{S_{n_l}}(y_i-a_j^{n_l})^2=\frac{1}{2}\sum_{j=1}^{S_{n_l}}\frac{\partial}{\partial z_i^{(n_l-1)}}(y_j-f(z_j^{n_l}))^2\\根据求导数的链式法则可以得到\\ =-\sum_{j=1}^{S_{n_l}}(y_j-f(z_j^{(n_l)}))\cdot \frac{\partial}{\partial z_i^{(n_l-1)}}f(z_j^{n_l})=-\sum_{j=1}^{S_{n_l}}(y_j-f(z_j^{(n_l)}))\cdot f'(z_j^{(n_l)})\cdot \frac{\partial z_j^{(n_l)}}{\partial z_i^{(n_l-1)}}\\根据公式(5)以及神经网络前向传播公式z_j^{n_l}=\sum_{k=1}^{S_{n_l-1}}f(z_k^{n_l-1})\cdot W_{kj}^{n_l-1}可以得到\\=\sum_{j=1}^{S_{n_l}}\delta_j^{(n_l)}\cdot \frac{\partial z_j^{(n_l)}}{\partial z_i^{n_l-1}}=\sum_{j=1}^{S_{n_l}}\delta_j^{n_l}\cdot \frac{\partial}{\partial z_i^{(n_l-1)}}(\sum_{k=1}^{S_{n_l-1}}f(z_k^{n_l-1})\cdot W_{kj}^{n_l-1})\\由于当且仅当k=i的时候\frac{\partial f(z_k^{(n_l-1)})}{\partial z_i^{(n_l-1)}}才为非0所以和式仅剩下一项，得到\\ =\sum_{j=1}^{S_{n_l}}\delta_j^{n_l}\cdot W_{ij}^{n_l-1}\cdot f'(z_i^{n_l-1})=(\sum_{j=1}^{S_{n_l}}W_{ij}^{n_l-1}\delta_j^{(n_l)})f'(z_i^{n_l-1}) \\
将得到的公式形式进一步整理，将n_{l-1}和n_{l}的关系替换为l和l+1的关系，得到：\\
\delta_i^{(l)}=(\sum_{j=1}^{s_{l+1}}W_{ij}^{(l)}\delta_j^{(l+1)})f'(z_i^{(l)}) \quad 公式(6)即得证
$$

**step-7 通过迭代形式的第$l$层第$i$个节点神经元上的$delta$残差，推导关于w&b的偏导数**

$$
结论1 \quad \frac{\partial}{\partial W_{ij}^{(l)}}J(W,b;x,y)=a_j^{(l)}\delta_i^{(l+1)}    \tag{7-1}
$$
$$
结论2 \quad \frac{\partial}{\partial b_{i}^{(l)}}J(W,b;x,y)=\delta_i^{(l+1)}    \tag{7-2}
$$

$$
公式(7-1)的推导\ 已知\frac{\partial}{\partial z_i^{(l)}}J(W,b;x,y)=\delta_i^{(l)}=公式(6)\\因此\ \frac{\partial}{\partial W_{ij}^{(l)}}J(W,b;x,y)=\frac{\partial J(W,b;x,y)}{\partial z_i^{(l)}} \cdot \frac{\partial z_i^{(l)}}{\partial W_{ij}^{(l)}}\\ 然而\frac{\partial z_i^{(l)}}{\partial W_{ij}^{(l)}}=a_j^{(l)} \ 所以公式即得证
$$

$$
公式(7-2）的推导\ 已知\frac{\partial}{\partial z_i^{(l)}}J(W,b;x,y)=\delta_i^{(l)}=公式(6)\\因此\ \frac{\partial}{\partial b_{i}^{(l)}}J(W,b;x,y)=\frac{\partial J(W,b;x,y)}{\partial z_i^{(l)}} \cdot \frac{\partial z_i^{(l)}}{\partial b_{i}^{(l)}}\\ 然而\frac{\partial z_i^{(l)}}{\partial b_{i}^{(l)}}=1 \ 所以公式即得证
$$

至此，我们就得到了误差函数关于神经网络层的中第$l$层第$i$个节点神经元和第$l$层第$j$个节点神经元的权重向量$W_{ij}^l$以及第$l$层第$i$个节点神经元的偏置$b_{i}^l$的导数。

### # method in pics
使用图片说明神经网络反向传播计算逻辑关系。
<!-- <center><img src="/img/in-post/vc/vc0.pdf" width="80%"></center> -->


### # reference
ufldl tutorial
