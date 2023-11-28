---
layout: post
title: "Parameter Optimizer"
subtitle: '常见机器学习参数优化器介绍'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-04-16 09:29 
lang: ch 
catalog: true 
categories: documentation
tags:
  - conv 
  - Time 2020
---
## Abstract
本文主要对于常见的几种优化器进行简介，一方面帮助在实际使用过程中优化器进行选择，另一方面方便以后进行知识回顾。本文主要参考了[这篇论文](https://arxiv.org/pdf/1609.04747.pdf)以及[这篇博客](https://www.cnblogs.com/guoyaohua/p/8780548.html)进行知识整理，并结合自己的理解，辅以图示，便于直观理解知识。

## Preparations
下面将要介绍的参数优化更新算法主要基于`gradient descent`算法，并且对于基础的`gd`算法进行了优化。`gd`算法：在目标函数$J(\theta)$中的参数空间(可行域)中，沿着目标函数$J(\theta)$的关于($w.r.t.$)参数$\theta$的梯度$\nabla_{\theta}J(\theta)$的相反方向来优化更新参数$\theta$，最终使得目标函数落入局部或全局最小值的过程。其中学习率$\eta$决定了参数$w.r.t(with\ respect\ to)$梯度的更新程度。

## GD variants
`BGD`、`SGD`以及`MBGD`是三种`gd`算法的变种，这三种算法在计算梯度数值时，可通过使用的数据样本量进行区分。计算一次梯度数值所使用的样本量越大，参数更新的准确性越高，但相应计算量越大，在具体应用过程中，不同方法选择需要进行权衡。

### BGD
`batch gradient descent`采用 ❮❮❮ $\theta=\theta-\eta\cdot\nabla_{\theta}J(\theta)$ ❯❯❯ 公式更新参数，每次计算梯度过程中需要将整个`batch`(数据集)加载进内存中，进行计算，计算耗时且容易爆内存，并且`bgd`不支持在数据集中添加新样本进行参数更新(无法$online\ learning$)，但是这种方式每次参数更新的方向一般来说最准确，对于凸问题，可以保证收敛到全局最优值，但是对于非凸问题，保证收敛到局部最优。

```python
for i in range(nb_epochs): # whole batch learning
    params_grad = evaluate_gradient(loss_function, data, params) 
    params = params - learning_rate * params_grad
```

### SGD
`stocastic gradient descent`相比`bgd`中一次采用整个`batch`的数据进行参数更新相比，`sgd`的方法使用单个样本$(x^{(i)},y^{(i)})$进行梯度计算：❮❮❮ $\theta=\theta-\eta\cdot\nabla_{\theta}J(\theta;x^{(i)};y^{(i)})$ ❯❯❯。这种使用单个样本的方法运行起来非常快，且支持$online\ learning$，但代价是每次梯度更新的方式可能不是最佳方向。在接近局部或者全局的极小值时，`sgd`方法可能会难以快速达到minima，梯度方向不准确，在极小值附近跳动，不过这种情况可以通过逐步减小的学习率进行改善。`sgd`的梯度更新存在一定的随机性，但也有可能从局部最小跳出来，转到更优的minima位置。

```python
for i in range(nb_epochs): 
    np.random.shuffle(data) 
    for example in data: # per example learning
        params_grad = evaluate_gradient(loss_function , example , params)
        params = params - learning_rate * params_grad
```

### MBGD
`mini-batch gradient descent`算法取`bgd`和`sgd`两家之长，采用`mini-batch`个数据样本来计算梯度：❮❮❮ $\theta=\theta-\eta\cdot\nabla_{\theta}J(\theta;x^{(i:i+n)};y^{(i:i+n)}),\quad batch\ size=n$ ❯❯❯。`mbgd`的梯度更新方式，减少了参数更新的方差，相比`sgd`参数更新更加平稳；使用矩阵化的批次参数更新计算库，能够加速运算，比`gd`更加快且内存占用小。一般`batch size`大小取$50\to 256$，可以根据实际经验进行选取。取两家之长，必染两家之短，`mbgd`和`sgd`一样，不能够保证在minima处的稳定性，对学习率设置要求高，和`gd`一样，需要根据炼丹炉的器型选择`batch size`大小，否则也时有内存溢出错误发生。

```python
for i in range(nb_epochs): 
    np.random.shuffle(data)
    for batch in get_batches(data, batch_size=50): # per mini-batch learning
        params_grad = evaluate_gradient(loss_function, batch, params) 
        params = params - learning_rate * params_grad
```
## Drawbacks of GD
`1` 基于`gd`的上述三种算法学习率$\eta$参数很难设置。$\eta$太小将使得模型参数收敛非常缓慢，并且如果陷入到一个不合理的`local minima`中，将难以跳跃出来寻找更优的极小值。$\eta$太大使得模型参数收敛加快，但是常常在极小值处剧烈震荡，难以平稳下来，甚至导致发散。<br>
`2` `gd`算法针对数据中包含的所有特征采用相同的学习率进行参数更新，如果数据是稀疏的或者不同的特征在数据集中出现频率有较大差异，我们可能需要对于较稀疏的特征、出现频率较低的特征采用较大的$\eta$，`gd`算法难以满足此类需求。<br>
`3` 对于`non-convex`问题中，`gd`算法容易落入到`sub-optimal minima`中，但是有[论文](https://arxiv.org/pdf/1406.2572.pdf)指出，模型优化的困难主要不是由于收敛到`local minima`引起的，而是由于陷入到`saddle points`中(这些鞍点通常由相同误差的梯度平稳空间组成)，这种空间结构，当前`saddle point`所有方向梯度均为0，导致`gd`甚至`sgd`算法难以逃出鞍点魔掌。

<center><img src="/img/in-post/optimizer/saddle.pdf" width="100%"></center>

## Adaptive algorithms
下面讨论的自适应的算法，相对于上面单纯基于梯度的算法，这些算法对于应用场景有更好的适应性，具备更好的性质，能够简化学习率参数的设置难度，能够处理鞍点，并且能够针对不同的参数设置不同的学习率以应对特征稀疏等难优化的情况。

### Prerequisites
`指数加权平均`：❮❮❮ $v_t=\beta v_{t-1}+(1-\beta)\theta_t$ ❯❯❯，$v_t$表示$t$时刻的滑动平均值，$\theta_t$表示$t$时刻的数值。则$v_t$可近似表示从$t$时刻向前$\frac{1}{1-\beta}$个参数的加权平均。<br> 
`1` 取$\beta=0.9$对上式进行展开：$v_{100}=\beta v_{99}+(1-\theta)\theta_{100}=0.9v_{99}+0.1\theta_{100}$ <br>
`2` 对于$v_{99}$继续进行展开，$v_{99}=0.9v_{98}+0.1\theta_{99}, v_{98}=0.9v_{97}+0.1\theta_{98}, \quad \dots$ <br>
`3` 则$v_{100}=0.1\times 0.9^0\theta_{100}+0.1\times 0.9^1\theta_{99}+0.1\times 0.9^2\theta_{98}+0.1\times 0.9^3 v_{97}+\cdots$ <br>
`4` 由于$\beta^{\frac{1}{1-\beta}}=0.9^{10}\approx 0.35$，因此可以近似的将指数加权平均看成是最近$\frac{1}{1-\beta}$个数的加权平均。<br>
`偏差修正`：在上述加权平均计算的过程中，在$t$很小的时候，按照上式计算得到的加权平均值和真实的数值相差特别大(可以用$\beta=0.98,t=2$进行验证)，因此，如果我们特别关注初始迭代步骤计算所得的加权平均数值的准确性，那么应采用修正的形式：<br>
`1` 修正后的形式：❮❮❮ $\frac{v_t}{1-\beta^t}=\beta v_{t-1}+(1-\beta)\theta_t$ ❯❯❯，在$t$很小时，$1-\beta^t$较小，使得计算得到的$v_t$数值接近真实均值，当$t$很大的时候，$1-\beta^t\approx 1$，和无修正的计算数值一致。<br>

### Momentum
`momentum`按照：❮❮❮ $v_t=\gamma v_{t-1}+\eta \nabla_{\theta}J(\theta), \quad \theta=\theta-v_t$ ❯❯❯ <br>
来计算经过指数滑动平均后的更新值，用来更新$\theta$参数，其中$\eta$是momentum系数，通常被设置成$0.9$，$\eta$系数设置的越大，表示当前参数更新数值越依赖当前样本计算得到的反传梯度，$\gamma$是指数滑动平均系数，一般可设置成$0.999$，表示当前梯度数值计算依赖从当前iteration向前$\frac{1}{1-0.999}=1000$个`mini-batch`梯度更新共同加权确定。因此$\eta$和$\gamma$是两个互相`拮抗`的参数。整个过程就像是一个`vodka酒鬼`走的每一步都带有前几步的未释放完的`动量`一样。

<center><img src="/img/in-post/optimizer/momentum.pdf" width="80%"></center>

### NAG
`Nesterov Accelerated Gradient`按❮❮❮$v_t=\gamma v_{t-1}+\eta \nabla_{\theta}J(\theta-\gamma v_{t-1}), \theta=\theta-v_t$❯❯❯ <br>
的方式获得参数$\theta$更新数值。对于`momentum`公式中$\nabla J(\theta) => \nabla J(\theta -\gamma v_{t-1})$，主要想法是为了避免：因为原来下坡积累的动量太大，遇到`上坡`的情况，不能即时刹车的情况。所以可以将`nag`看成是一种修正过的`momentum`梯度优化器。但是，有的时候我们想要对没轮次迭代中的每个参数设置不同(自适应)的学习率，adagrad能够满足这类要求。

### Adagrad
`Adaptive gradient algorithm`按照 
❮❮❮ $\theta_{t+1,i}=\theta_{t,i}-\frac{\eta}{\sqrt{G_{t,ii}+\epsilon}}\cdot\nabla_{\theta_t}J(\theta_{t,i})$ ❯❯❯  <br>
来计算自适应学习率参数更新数值。其中`adagrad`算法使用$\frac{\eta}{\sqrt{G_{t,ii}+\epsilon}}$代替之前固定的学习率$\eta$，`adagrad`对于每个时刻$t$的模型中的每个参数$\theta_i$，根据之前的所有时刻的参数更新梯度累加和，动态设置学习率，实施：之前参数更新`动量`越大，此时此参数的学习率越低的调整策略。其中学习率设置为：❮❮❮ $G_{t,ii}=[\sum_0^t( \nabla_{\theta_t}J(\theta_{t,0}))^2, \sum_0^t( \nabla_{\theta_t}J(\theta_{t,i}))^2, \cdots , \sum_0^t( \nabla_{\theta_t}J(\theta_{t,n}))^2 ]$ ❯❯❯，$G_{t,ii}$是对角矩阵。由于$G_{t,ii}$是一个方阵，$\nabla_{\theta_t}J(\theta_{t,i})$是一个向量，因此`adagrad`可以改写成`element-wise`元素相乘的形式$\odot$。

`adagrad`方法优化的方法使学习率适应参数，对累计梯度之和(动量)越小的参数执行较大的更新，对累计梯度之和(动量)越大的参数进行较小的更新，因此，非常适合处理稀疏数据。这种优化方法不需要精心设计各个轮次迭代的学习率，一般选取$0.01$作为初始值即可，但是，它由于梯度累加的关系，导致学习率分母越来越大，容易导致学习率随训练越来越小，参数基本不更新。为了解决`adagrad`学习率很快下降的问题，提出了`adadelta`和`RMSprop`优化器。

### Adadelta
`adadelta`是`adagrad`的扩展，旨在降低其激进的，单调降低的学习率。`adadelta`不会累计所有过去($0:t$)的平方梯度，而是将累计过去的梯度的窗口限制为某个固定大小$(t-w):t$。

**Idea-1** <br>
方法1按照
❮❮❮ $$\theta_{t+1}=\theta_{t}-\frac{\eta}{\sqrt{E[g^2]_t+\epsilon}}\nabla_{\theta_t}J(\theta_{t,i}) =\theta_t-\frac{\eta}{RMS[g]_t} \nabla_{\theta_t}J(\theta_{t,i})$$ ❯❯❯
来控制学习率，并完成参数更新。 其中：❮❮❮ $\sqrt{E[(\nabla_{\theta_t}J(\theta_{t,i}))]^2 +\epsilon}=RMS[\nabla_{\theta_t}J(\theta_{t,i})]$ ❯❯❯，且RMS为`root mean squared`(梯度均方根)的简写。公式中使用梯度平方的`指数滑动平均`值来计算上式中的期望($E$)：❮❮❮ $E[\nabla_{\theta_t}J(\theta_{t,i})^2]=\gamma E[\nabla_{\theta_{t-1}}J(\theta_{t-1,i})^2]+(1-\gamma)\nabla_{\theta_t}J(\theta_{t,i})^2$ ❯❯❯。

**From Idea-1 to Idea-2** <br>
idea-1：一阶近似方法`sgd`和`momentum`正比于梯度，$\Delta \theta \propto g\propto \frac{\partial f}{\partial \theta} \propto \frac{1}{\theta}$ <br>
idea-2：二阶近似方法采用类似`牛顿法`的方式，利用了hessian矩阵的信息，正比于hessian矩阵的逆乘梯度：$\Delta \theta \propto H^{-1}g\propto \frac{\frac{\partial f}{\partial \theta}}{\frac{\partial^{2}f}{\partial \theta^{2}}}\propto \frac{\frac{1}{\theta}}{\frac{1}{\theta}*\frac{1}{\theta}}\propto \theta$。一阶方式最终梯度正比于参数大小的倒数，因此，会导致参数值越大的，其反传梯度越小，越难以更新；而二阶方式最终梯度正比于参数大小，因此梯度能够自适应的跟随参数数量级进行变化，核心思想是对hessian逆阵的近似。

**Idea-2** <br>
方法2按❮❮❮ $$\Delta \theta_t \approx  H^{-1} g = \frac{\frac{\partial f}{\partial \theta}}{\frac{\partial^{2}f}{\partial \theta^{2}}} =\frac{1}{\frac{\partial^{2}f}{\partial \theta^{2}}}\cdot \frac{\partial f}{\partial \theta}=\frac{1}{\frac{\partial^{2}f}{\partial \theta^{2}}}\cdot g_{t}=\frac{\Delta \theta}{\frac{\partial f}{\partial \theta}} \cdot g_t\approx -\frac{RMS[\Delta \theta]_{t-1}}{RMS[g]_{t}}\cdot g_t$$ ❯❯❯ <br>
其中$g_t=\nabla_{\theta_t} J(\theta_{t,i})$来设定学习率，并按照❮❮❮ $$\theta_{t+1}=\theta_t - \frac{RMS[\Delta \theta]_{t-1}}{RMS[g]_{t}}\cdot \nabla_tJ(\theta_{t,i})$$ ❯❯❯，其中 ❮❮❮ $$RMS[\Delta\theta]_t=\sqrt{E[\Delta\theta^2]+\epsilon}$$ ❯❯❯ 完成参数更新。

<center><img src="/img/in-post/optimizer/adadelta2.pdf" width="90%"></center>

### RMSprop
`rmsprop`和`adadelta`部分介绍的第一种情况(**Idea-1**)是相同的，此处不在赘述。根据hinton建议，**Idea-1**方法中：滑动平均系数$\gamma=0.9$，学习率$\eta=0.001$

### Adam
`Adaptive Moment Estimation`相当于`RMSprop+Momentum`方法。<br>
`adam`算法按照 ❮❮❮ $\theta_t=\theta_{t-1}-\alpha\cdot\frac{\hat{m_t}}{\sqrt{\hat{v_t}}+\epsilon}$ ❯❯❯，进行学习率的设置，以及进行参数的更新。具体的算法见下图：

<center><img src="/img/in-post/optimizer/adam.pdf" width="90%"></center>

### AdaMax
一种修正的adam，通过❮❮❮ $v_t=\beta_2^{\infty}v_{t-1}+(1-\beta_2^{\infty})\mid g_t\mid^{\infty}=max(\beta_2\cdot v_{t-1},\ \mid g_t\mid)$ ❯❯❯ 以及 ❮❮❮ $\theta_{t+1}=\theta_t-\frac{\eta}{v_t}\hat{m_t}$ ❯❯❯进行参数更新。<br>
首先回忆`adam`的一般形式 $v_t=\beta_2^pv_{t-1}+(1-\beta_2^p)\mid g_t\mid^p,\quad  if\ p=1 => adam$，一般来说，$l^p$范数$p$值越大越不稳定，但是$l^\infty$范数却可以保持一定的稳定性，以此作为依据，构建了`adamax`。

### Nadam
`adam`优化器可以看作是`RMSprop`和`momentum`优化器的结合，根据之前的介绍，`nag`优化器对于`momentum`做了一些改进(能够一定程度避免动量累计导致的上坡不减速的情况)，因此将`RMSprop`和`nag`结合，产生`Nadam`。基于上述介绍的基础，`nadam`的改进思路流程如下图所示：

<center><img src="/img/in-post/optimizer/nadam.pdf" width="80%"></center>

## Comparison
各个算法在鞍点`saddle point`处的表现如下图，`sgd`算法无法逃离鞍点，导致其不能想起他算法一样找到更优的最小值。

<center><img src="/img/in-post/optimizer/optimizer_in_saddle.gif" width="60%"></center>

各个算法的收敛速度图如下，由快到慢的顺序大概是黄(`adadelta`) - 紫(`NAG`) - 绿(`momentum`) - 黑(`rmsprop`) - 蓝(`adagrad`) - 红(`sgd`)。

<center><img src="/img/in-post/optimizer/speed_of_convergence.gif" width="60%"></center>

## Reference
> https://arxiv.org/pdf/1609.04747.pdf <br>
> https://www.cnblogs.com/guoyaohua/p/8780548.html


> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内