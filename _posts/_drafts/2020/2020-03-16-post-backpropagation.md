---
layout: post
title: "Backpropagation algorithm"
subtitle: 'bp算法原理解析|matlab实现'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-03-16 22:26 
lang: ch 
catalog: true 
categories: documentation
tags:
  - optimization algorithm
  - Time 2020
---

## Introduction
本文主要对于神经网络的反向传播算法进行总结，并以此为基础完成全连接网络的[fc_bp_matlab]()程序构建，以及经典的卷积神经网络程序[cnn_bp_matlab]()的构建。

## Preparations
`样本数据` 假设我们有一个包含$m$个样本的数据集$\{(x^{(1)},y^{(1)}),(x^{(2)},y^{(2)}),\cdots, (x^{(m)},y^{(m)})\}$，$x$表示数据样本，$y$表示相应的label。

## Method
**step-1 构建单个样本的代价函数(loss function)** <br>
$h\_{W,b}(x)$是神经网络的输出，$y$是数据集中所给出的label，不妨设代价函数为平方误差代价函数：

$$
J(W,b;x,y)=\frac{1}{2}||h_{W,b}(x)-y||^2    \tag{1}
$$

**step-2 构建m个样本数据的代价函数(loss function)** <br>
在公式$(1)$的基础上，构建带有神经网络正则化的误差函数。主要的符号定义如下：<br> 
`1` $i$表示总共$m$个样本的第$i$样例。`2` $l$表示神经网络共$n_l$层神经网络的第$l$层。`3` $s_l$表示神经网络第$l$层中共有$s_l$个节点神经元。`4` $s_{l+1}$表示第$l+1$层共有$s_{l+1}$个节点神经元。`5` $W_{ij}^l$表示第$l$层第$i$个节点神经元到第$l+1$层第$j$个神经元的权值，同理，$W_{kj}^l$表示第$l$层第$k$个节点神经元到第$l+1$层第$j$个神经元的权值。

$$
J(W,b)=\frac{1}{m}\sum_{i=1}^mJ(W,b;x^{(i)},y^{(i)})+\frac{\lambda}{2}\sum_{l=1}^{n_l-1}\sum_{i=1}^{s_l}\sum_{j=1}^{s_{l+1}}(W_{ij}^{(l)})^2    \tag{2-1}
$$
$$
=\frac{1}{m}\sum_{i=1}^m(\frac{1}{2}||h_{W,b}(x^{(i)})-y^{(i)}||^2)+\frac{\lambda}{2}\sum_{l=1}^{n_l-1}\sum_{i=1}^{s_l}\sum_{j=1}^{s_{l+1}}(W_{ij}^{(l)})^2    \tag{2-2}
$$

**step-3 构建神经网络参数w&b更新公式** <br>
根据反向传播算法是最优化理论中梯度下降法在神经网络中的应用，参数$\alpha$表示参数学习率。

$$
W_{i,j}^{(l)}=W_{i,j}^{(l)}-\alpha \frac{\partial}{\partial W_{i,j}^{(l)}}J(W,b), \quad  b_i^{(l)}=b_i^{(l)}-\alpha \frac{\partial}{\partial b_i^{(l)}}J(W,b)    \tag{3}
$$

**step-4 先求loss-function关于w&b参数的偏导数** <br>

$$
\frac{\partial}{\partial W_{i,j}^{(l)}}J(W,b)=\frac{1}{m}\sum_{i=1}^{m}\frac{\partial}{\partial W_{i,j}^{(l)}}J(W,b;x^{(i)},y^{(i)})+\lambda W_{i,j}^{(l)}    \tag{4-1}
$$
$$
\frac{\partial}{\partial b_i^{(l)}}J(W,b)=\frac{1}{m}\sum_{i=1}^m\frac{\partial}{\partial b_i^{(l)}}J(W,b;x^{(i)},y^{(i)})    \tag{4-2}
$$

**step-5 以单个数据样本$(x,y)$的对于w&b的偏导数为例，求解bp第$n_l$层残差$delta$通用形式** <br>
下面公式中，为了方便书写，$h_{W,b}(x)$简记为$a_i^{n_l}$，表示最后一层的激活值，令$z_i^{n_l}$表示最后一层的待激活值，则有如下关系成立：$a_i^{n_l}=f(z_i^{n_l})=h_{W,b}(x)$成立，其中$f$表示激活函数。

$$
结论\quad\delta_i^{n_l}=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}||y-h_{W,b}(x)||^2=-(y_i-a_i^{n_l})\cdot f'(z_i^{n_l})    \tag{5}
$$

$$
公式(5)的推导:\ \delta_i^{n_l}=\frac{\partial}{\partial z_i^{n_l}}J(W,b;x,y)=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}||y-h_{W,b}(x)||^2\\ =\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}\sum_{j=1}^{S_{n_l}}(y_i-a_j^{(n_l)})^2=\frac{\partial}{\partial z_i^{n_l}}\frac{1}{2}\sum_{j=1}^{S_{n_l}}(y_i-f(z_j^{(n_l)}))^2\\ 上面的式子只有当j=i的时候对z_i^{(n_l)}求解偏导数才有数值 \ 否则为0 \ 所以上面的式子等价于 \\  -(y_i-f(z_i^{(n_l)}))\cdot f'(z_i^{(n_l)})=-(y_i-a_i^{(n_l)})\cdot f'(z_i^{(n_l)}) \quad 公式(5)即得证
$$

**step-6 根据链式求导法则 求解神经网络第$l$层第$i$个节点神经元上的bp残差的通式**

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

## Method in pics
使用图片说明神经网络反向传播计算逻辑关系。
<!-- <center><img src="/img/in-post/vc/vc0.pdf" width="80%"></center> -->


## Reference
> UFLDL tutorial

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
