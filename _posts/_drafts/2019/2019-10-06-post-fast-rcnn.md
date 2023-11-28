---
layout: post
title: "Fast-RCNN"
subtitle: '原理流程解析|roi_pooling解析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-10-06 13:20 
lang: ch 
catalog: true 
categories: documentation
tags:
  - object detection
  - Time 2019
---

## Background

经典的[RCNN](/documentation/2019/09/29/post-rcnn/)虽然是RCNN系列的开山之作，但是存在一些性能上的问题：如训练分为多个阶段进行，不能够end-to-end进行训练，中见提取出来的feature需要开辟空间进行保存；对于应用[selective search](/documentation/2019/10/01/post-region-proposal/)产生的2000个proposal bbox都分别进行alexnet进行分类训练，以及最后产生confidence的时候需要训练`1 vs rest`的20+1个SVM分类器，从时间和空间上花销巨大；另外，RCNN中对于每个proposal都进行卷积提取特征，由于proposal bbox存在大量的重叠，因此在训练测试过程会出现很多冗余计算，[SPPnet](https://arxiv.org/pdf/1406.4729.pdf)就是为了解决重叠bbox导致冗余卷积计算而产生的，但是训练分多个阶段，空间开销大的问题不能被解决。Fast-RCNN的思路非常巧妙的解决上述两个问题，为后面的其他著名的检测算法的提出，给予了重要的启发。

## Introduction
fast-RCNN舍弃了先进行region-proposal再分别提取特征的模式，而是通过roi_pooling的结构完成了region proposal的引入，最大程度的共用了神经网络前几层的`特征提取`的共性，减少了重复计算；另外，使用了softmax代替了[RCNN](/documentation/2019/09/29/post-rcnn/)和[SPPnet](https://arxiv.org/pdf/1406.4729.pdf)中`1 vs rest`的SVM。fast-RCNN的结构框架图如下，整个结构相比RCNN要更加简洁：

<center><img src="/img/in-post/fast_rcnn/structure.png" width="60%"></center>

fast-RCNN中的核心思想在于roi_pooling层，这一层主要完成了两件事情：第一，这一层是将region proposal的结果融合进backbone network(Alexnet)的层；第二，这个操作使得对于不同大小的region proposal bbox能够产生相同尺度的输出，方便后面接入全连接层进行分类以及bbox变换回归。上面的图片来自fast-RCNN论文，其中关于通过[selective search](/documentation/2019/10/01/post-region-proposal/)方式产生的region proposal如何作用到conv_5得到的 feature map不是很清晰，为此，绘制更加详细的流程图方便理解：

<center><img src="/img/in-post/fast_rcnn/fast_rcnn_flow.png" width="90%"></center>

## ROI Pooling
下面介绍fast-RCNN文章中的核心部分`roi pooling`。这个结构实现的功能很简单，但是如果想要在tensorflow里面使用，需要regist新的operation，然后再使用，否则直接使用tensorflow实现会严重影响模型的训练速度。`tensorflow`官方已经为我们实现了这个函数`tf.image.crop_and_resize()`，非官方的tensorflow实现，我推荐[这个repo](https://github.com/twistfatezz/roi-pooling)中的实现方式。

Ps:我曾经使用tensorflow实现了一个spatial pyramid pooling的功能，[可以参考这个repo](https://github.com/twistfatezz/dongh_spatial_pyramid_structure)，但是发现，由于这个操作无法完全并行(自己太菜)，所以导致应用的时候，速度很慢ToT。

关于`roi pooling`层的作用，使用下面的图进行描述，我的所有的对于`roi pooling`层的理解都画在这张图中了：

<center><img src="/img/in-post/fast_rcnn/roi_pooling.png" width="90%"></center>

图中有几个重要的点需要强调：第一：roi pooling操作接收的input包含两个部分(region proposal + conv5 产生的feature map)，其中feature map的输入batch_size不一定是1，为了能够将这个op的流程绘制清晰，选取batch_size=1。第二：输入feature map的通道数=2，对于[selective search](/documentation/2019/10/01/post-region-proposal/)产生的proposal会作用于每一个channel上。第三：如图，图中绘制的是proposal的bbox为4的情况。第四(最重要的一步)：注意输出的tensor的形状中`batch_size`参数为`1x4`，解释：$$batch\_size = input\_batch\_size \times num\_bbox\_chosen\_from\_each\_sample\_of\_the\_input\_batch$$。w和h参数是按照蓝线进行分割的出的每一块进行pooling的并进行flatten操作结果。

## Model Analysis 
本小节用于结合fast-RCNN代码，按照模型的inference顺序，结合相应tensorflow代码进行模型流程逐段分析。这样能够对于fast-RCNN模型的架构以及数据流有更加准确清晰的认识。

<span id="return_structure"> </span>
### structure 
**上面的两张图片已经完整的显示了fast-RCNN的主要框架结构，此处不再赘述。** [[Code]](#model structure)

<span id="return_loss"> </span> 
### loss function 
**Multi-task loss. A Fast R-CNN network has two sibling output layers. The first outputs a discrete probability distribution(perRoI), $$p=(p0,...,pK)$$,over $K+1$ categories. As usual, $p$ is computed by a softmax over the $K+1$ outputs of a fully connected layer. The second sibling layer outputs bounding-box regression offsets, $$t_k=(t_k^x, t_k^y, t_k^w, t_k^h)$$, for $x,y,w,h$ each of the $K$ object classes, indexed by $k$. We use the parameterization for $tk$ given in [RCNN](/documentation/2019/09/29/post-rcnn/), in which $tk$ specifies a scale-invariant translation and log-space height/width shift relative to an object proposal.** 

**Each training RoI is labeled with a ground-truth class $u$ and a ground-truth bounding-box regression target $v$. We use a multi-task loss $L$ on each labeled RoI to jointly train for classification and bounding-box regression:**  

$$
L\left(p,u,t^{u},v\right)=L_{\mathrm{cls}}(p, u)+\lambda[u \geq 1] L_{\mathrm{loc}}\left(t^{u}, v\right)
$$

「**confidence loss**」
**confidence loss是普通的交叉熵损失，对于每个input img中的64个bbox(positive & negative)都用于计算。in which $$L_{cls}(p, u)$$ is log loss for true class u.** 

$$
L_{cls}(p, u)=−\log p_u
$$

「**location loss**」
**bbox regression仅仅在positive bbox和ground truth bbox进行计算，negative bbox是不予考虑的。The second task loss, Lloc, is defined over a tuple of true bounding-box regression targets for class $$u,v=(v_x, v_y, v_w, v_h)$$, and a predicted tuple $$t^u = (t^u_x, t^u_y, t^u_w, t^u_h)$$, again for class $u$. The Iverson bracket indicator function(指示函数) $[u ≥ 1]$ evaluates to 1 when $u ≥ 1$ and 0 otherwise. By convention the catch-all background class is labeled $u = 0$.**

$$
L_{\mathrm{loc}}\left(t^{u}, v\right)=\sum_{i \in\{\mathrm{x}, \mathrm{y}, \mathrm{w}, \mathrm{h}\}} \operatorname{smooth}_{L_{1}}\left(t_{i}^{u}-v_{i}\right)
$$

**其中，location loss的包裹函数为smooth-L1 loss：**

$$
\operatorname{smooth}_{L_{1}}(x)=\left\{\begin{array}{ll}{0.5 x^{2}} & {\text { if }|x|<1} \\ 
{|x|-0.5} & {\text { otherwise }}\end{array}\right.
$$

[[Code]](#loss)

## Supplementary Analysis
**Mini-batch for total network**
> During fine-tuning, each SGD mini-batch is constructed from $N=2$ images, chosen uniformly at random (as is common practice, we actually iterate over permutations of the dataset). <br>
> `explain:` finetune的时候整个网络的input的batch_size=2，一个batch里面有两张原始图像。<br>

**mini_batch for conf&reg fc network**
> We use mini-batches of size $R=128$, sampling 64 RoIs from each image. <br>
> `explain:` 对于`roi pooling`的部分(从每个sample产生的feature map中选取64个bounding box，总共$2\times 64$个bbox输入到用于生成confidence & regression的sibling fc nets中)。<br>

**每个输入图像中的64个region proposal bbox如何选择**
> As in [9], we take 25% of the RoIs from object proposals that have intersection over union (IoU) overlap with a ground-truth bounding box of at least 0.5. These RoIs comprise the examples labeled with a foreground object class, i.e. u >= 1. The remaining RoIs are sampled from object proposals that have a maximum IoU with ground truth in the interval [0.1, 0.5), following [11]. These are the background examples and are labeled with u = 0. The lower threshold of 0.1 appears to act as a heuristic for hard example mining [8]. During training, images are horizontally flipped with probability 0.5. No other data augmentation is used. <br>
> `explain:` 每个输入图像(总共有两个)中，需要选取64个bbox，其中正样本16个，负样本48个。正样本(u>=1)iou>0.5中sample，负样本(u=0)在[0.1,0.5)的iou中sample。

### Truncated SVD for faster detection
### Do SVMs outperform softmax?
### Are more proposals always better?

## Code
<span id="model structure"> </span> 
### structure
```python
def maxPoolLayer(self, x, kHeight, kWidth, strideX, strideY, name, padding="SAME"):
    return tf.nn.max_pool(x, ksize=[1, kHeight, kWidth, 1],
                        strides=[1, strideX, strideY, 1], padding=padding, name=name)

def LRN(self, x, R, alpha, beta, name=None, bias=1.0):
    return tf.nn.local_response_normalization(x, depth_radius=R, alpha=alpha,
                                            beta=beta, bias=bias, name=name)

# group=2 对应Alexnet中上下两个部分
def convLayer(self, x, kHeight, kWidth, strideX, strideY, featureNum, name, padding="SAME", groups=1): 
    channel = int(x.get_shape()[-1])  # 获取channel
    conv = lambda a, b: tf.nn.conv2d(a, b, strides=[1, strideY, strideX, 1], padding=padding)  # 定义卷积的匿名函数
    with tf.variable_scope(name) as scope:
        w = tf.get_variable("w", shape=[kHeight, kWidth, channel / groups, featureNum])
        b = tf.get_variable("b", shape=[featureNum])
        tf.losses.add_loss(self.regularizer(w))
        xNew = tf.split(value=x, num_or_size_splits=groups, axis=3)  # 划分后的输入和权重
        wNew = tf.split(value=w, num_or_size_splits=groups, axis=3)

        featureMap = [conv(t1, t2) for t1, t2 in zip(xNew, wNew)]  # 分别提取feature map
        mergeFeatureMap = tf.concat(axis=3, values=featureMap)  # feature map整合
        out = tf.nn.bias_add(mergeFeatureMap, b)
        return tf.nn.relu(tf.reshape(out, mergeFeatureMap.get_shape().as_list()))  # relu后的结果

def build_network(self, images, class_num, is_training=True, keep_prob=0.5, scope='Fast-RCNN'):
    self.conv1 = self.convLayer(images, 11, 11, 4, 4, 96, "conv1", "VALID")
    lrn1 = self.LRN(self.conv1, 2, 2e-05, 0.75, "norm1")
    self.pool1 = self.maxPoolLayer(lrn1, 3, 3, 2, 2, "pool1", "VALID")
    self.conv2 = self.convLayer(self.pool1, 5, 5, 1, 1, 256, "conv2", groups=2)
    lrn2 = self.LRN(self.conv2, 2, 2e-05, 0.75, "lrn2")
    self.pool2 = self.maxPoolLayer(lrn2, 3, 3, 2, 2, "pool2", "VALID")
    self.conv3 = self.convLayer(self.pool2, 3, 3, 1, 1, 384, "conv3")
    self.conv4 = self.convLayer(self.conv3, 3, 3, 1, 1, 384, "conv4", groups=2)
    self.conv5 = self.convLayer(self.conv4, 3, 3, 1, 1, 256, "conv5", groups=2)

    # this implementation is based on https://github.com/deepsense-ai/roi-pooling 
    # tensorflow official implementation: tf.image.crop_and_resize()
    self.roi_pool6 = roi_pooling(self.conv5, self.rois, pool_height=6, pool_width=6)

    with slim.arg_scope([slim.fully_connected, slim.conv2d], activation_fn=nn_ops.relu,
                        weights_initializer=tf.truncated_normal_initializer(0.0, 0.01),
                        weights_regularizer=slim.l2_regularizer(0.0005)):

        flatten = slim.flatten(self.roi_pool6, scope='flat_32')
        self.fc1 = slim.fully_connected(flatten, 4096,  scope='fc_6')
        drop6 = slim.dropout(self.fc1, keep_prob=keep_prob, is_training=is_training, scope='dropout6',)
        self.fc2 = slim.fully_connected(drop6,  4096,  scope='fc_7')
        drop7 = slim.dropout(self.fc2, keep_prob=keep_prob, is_training=is_training, scope='dropout7')
        cls = slim.fully_connected(drop7, class_num, activation_fn=nn_ops.softmax ,scope='fc_8')
        bbox = slim.fully_connected(drop7, (self.class_num-1)*4,
                                    weights_initializer=tf.truncated_normal_initializer(0.0, 0.001),
                                    activation_fn=None ,scope='fc_9')
    return cls,bbox
```
[[Return]](#return_structure)

<span id="loss"> </span> 
### loss function
```python
def loss_layer(self, y_pred, y_true, box_pred):
    # confidence loss
    cls_pred = y_pred
    cls_true = y_true[:, :self.class_num]
    cls_pred /= tf.reduce_sum(cls_pred, reduction_indices=len(cls_pred.get_shape()) - 1, keep_dims=True)
    cls_pred = tf.clip_by_value(cls_pred, tf.cast(1e-10, dtype=tf.float32), tf.cast(1. - 1e-10, dtype=tf.float32))
    cross_entropy = -tf.reduce_sum(cls_true * tf.log(cls_pred), reduction_indices=len(cls_pred.get_shape()) - 1)
    cls_loss = tf.reduce_mean(cross_entropy)
    tf.losses.add_loss(cls_loss)
    tf.summary.scalar('class-loss', cls_loss)

    # bbox loss
    bbox_pred = box_pred
    bbox_ture = y_true[:, self.class_num:]
    mask = tf.tile(tf.reshape(cls_true[:, 1], [-1, 1]), [1, 4])
    for cls_idx in range(2, self.class_num):
        mask =tf.concat([mask, tf.tile(tf.reshape(cls_true[:, int(cls_idx)], [-1, 1]), [1, 4])], 1)
    bbox_sub =  tf.square(mask * (bbox_pred - bbox_ture))
    bbox_loss = tf.reduce_mean(tf.reduce_sum(bbox_sub, 1))
    tf.losses.add_loss(bbox_loss)
    tf.summary.scalar('bbox-loss', bbox_loss)
```
[[Return]](#return_loss)

## Reference
> A forked repo(roi_pooling in tensorflow) -> https://github.com/twistfatezz/roi-pooling <br>
> 上面repo的主页 https://deepsense.ai/region-of-interest-pooling-explained/ <br>
> 这篇blog对于caffe版的roi_pooling分析的很详细 -> https://blog.csdn.net/jiongnima/article/details/80016683 <br>
> code repo -> https://github.com/Liu-Yicheng/Fast-RCNN

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内