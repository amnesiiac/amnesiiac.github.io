---
layout: post
title: "Stereo RCNN"
subtitle: '3D object detection in stereo imagery'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-11-21 17:28 
lang: ch 
catalog: true 
categories: documentation
tags:
  - object detection 
  - Time 2019
---
## Abstract
本文要介绍的是[Stereo R-CNN based 3D Object Detection for Autonomous Driving](https://arxiv.org/pdf/1902.09738.pdf)。这篇文章通过使用`sparse`，`dense`，`semantic`，`geometry` 四个类别的信息(数据)实现了在`stereo imagery`场景下的三维目标识别任务。这个篇文章的特点是，不需要图像深度信息的输入，无须通过训练过程中使用3d-supervision方式指导模型生成深度，直接使用双目相机拍摄的left-right图像做3D检测，并且在KITTI数据集上达到SOTA效果。

## Introduction
三维目标检测的问题可以分成如下三个类别：<br>
**(1)** LiDAR-based，近期被研究的较多，基本是自动驾驶所必须的；缺点：成本高，感知范围较短(100m左右)，图像质量较差(lidar检测线比较少获取的信息有限)。 <br>
**(2)** Monocular-based，使用单眼相机的低成本方案；其中深度信息通过场景中的语义属性和对象大小等进行预测。缺点：推断的depth信息准确性有待质疑，尤其是对于被遮挡的物体的深度推测。<br>
**(3)** Stereo-based，和stereo cam相比LiDAR cam价格低廉，且能够达到comparable的深度估测精确度，另外stereo cam的感知范围取决于焦距和基线，因此具有提供更大范围感知的潜在能力。<br>


## Stereo R-CNN Network 
首先给出论文中的结构框图。通过这个图可以看出，整个结构是一种改造的[mask-rcnn](/documentation/2019/10/16/post-mask-rcnn/)的结构，有两个改进点：第一，将rpn改成了针对stereo cam的left-right图像的stereo rpn，第二，将mask-rcnn中后面回归的参数进行调整，使其适应双目3d重建问题需要。
<center><img src="/img/in-post/stereo_rcnn/structure.pdf" width="80%"></center>

### Stereo RPN
rpn用于分类和回归的不同任务相关的bbox如下图。
<center><img src="/img/in-post/stereo_rcnn/gt_bbox.pdf" width="80%"></center>

**objectness classification** <br>
取$$left\_gt\_bbox\cup right\_gt\_bbox=union\_bbox$$作为目标框 <br>
$$positive\_anchor:$$ $$iou(anchor,union\_bbox)>0.7$$ <br>
$$negtive\_anchor: iou(anchor,union\_bbox)<0.3$$ <br>

**stereo bbox regression** <br>
左边右边的bbox分别有自己的水平位置和宽度，但是共享一组垂直位置和高度(左目和右目图像在垂直方向已经经过对齐)
$$left\_gt\_bbox$$和$$right\_gt\_bbox$$均作为目标框 <br>
待回归参数：$$\left [u,w,u',w',v,h \right]$$，<br>
参数含义：$$\left[ left\_bbox\_x, left\_bbox\_width, right\_bbox\_x, right\_bbox\_width, bbox\_y, height \right]$$ <br>

最后分别在左侧和右侧roi上使用NMS以减少冗余，训练时左右两侧nms取top_2000，测试时候，nms取top_300。

### Stereo R-CNN

**[Stereo Regression]**
stereo reg branch中分别预测4个量：<br>
**(1)** object_class <br> 

**(2)** stereo_bbox，其中正负样本的定义为：<br>
正样本: $$iou(gt\_left\_bbox,left\_roi)>0.5 \; \& \& \; iou(gt\_right\_bbox,right\_roi)>0.5$$ <br>
负样本: $$max(iou(gt\_left\_bbox,left\_roi), iou(gt\_right\_bbox,right\_roi)) \in [0.5,0.1)$$

**(3)** dimension，先统计平均的尺寸作为基准值，然后网络回归的是相对量 <br>

**(4)** viewpoint_angle，如下图所示 <br>
<center><img src="/img/in-post/stereo_rcnn/angle.pdf" width="40%"></center>
$\theta$为相机坐标系下的朝向角(orientation)，$\beta$为相机中心点下的方位角(azimuth)，viewpoint_angle不是$\theta$不能够通过相机拍摄图像中直接观察得到，回归的量是视野角(viewpoint angle)，定义为$$\alpha=\theta+\beta$$，其中 $$\beta=arctan(−\frac{x_c}{z_c})$$$$(x,z是摄像机坐标系的数值)$$，并且考虑到连续性，回归量为 $$\left[ sin\alpha,cos\alpha \right ]$$。关于为什么要回归视野角而不是朝向角，参见原论文中的描述：

> In some cases where less than two side-surfaces can be completely observed and no perspective keypoint up (e.g., truncation, orthographic projection), the orientation and dimensions are unobservable from pure geometry constraints. We use the viewpoint angle α to compensate the unobservable states. <br> [`explain:`]即在某些情况下，只能看到少于两个侧面的物体，此时朝向角无法计算，因此通过回归在3d坐标中的视野角(在2d图像中观察不到)是有必要的。



**[Keypoint Prediction]**
只有双目图像中左边的图像对应的网络中的roi用于产生相应的图像的keypoints。
<center><img src="/img/in-post/stereo_rcnn/keypoint.pdf" width="60%"></center>

**(1) 3d semantic keypoints** 世界坐标系中的汽车中的四个角点 <br>
**(2) boundary keypoints** 图像坐标系中汽车的2d_bbox的左下和右下两个角点 <br>
**(3) perspective keypoints** 透视关键点，图像坐标系中位于两个boundary keypoints之间的反应物体3d透视信息的点 <br>
**(4) details** 类似mask-rcnn，在keypoint检测用的$6\times 28\times 28$的feature map的基础上，因为关键点只有图像坐标$u$方向才提供了额外的信息，所以对上述feature map中每列进行累加，最终输出$6\times 28$的向量。前面的4个通道($4\times 28$)代表了4个世界坐标系中的`3d semantic keypoints`投影到$u$坐标下形成一个`perspective keypoint`的概率，对$4\times 28$应用softmax就得到了是哪一个`semantic keypoint`产生了`perspective keypoint`以及产生的`perspective keypoint`的$u$方向坐标，后2个通道($2\times 28$)代表该$u$坐标是左右`boundary keypoints`的概率，分别对每个$1\times 28$应用softmax，就可以得到两个边缘角点的坐标。

### Dense 3D Box Alignment
这一部分介绍利用已知的2d信息获取3d_bbox的方式。3d_bbox状态可以用$$x=\left \{x,y,z,\theta \right\}$$和$w,h,l$表示，分别表示3d中心位置和相机坐标系下的朝向角。那么给定左右2d_bbox，perspective_bbox和dimension，可以通过最小化2d_bbox和keypoint和3d_bbox产生的投影之间的误差来求解相应的模型参数。

立体框和透视关键点中提取了七个测量值：$$z=\left \{u_l，v_t，u_r，v_b，u'_l，u'_r，u_p \right \}$$，分别代表left_2d_bbox的left，top，right，bottom的坐标，right_2d_bbox的左右边缘点perspective_keypoint的$u$方向坐标。给定透视关键点，结合2d_bbox中的边缘信息，可以推断出3d_bbox和2d_bbox之间的关系(如下图)。
<center><img src="/img/in-post/stereo_rcnn/projection.pdf" width="80%"></center>

用$b$代表stereo cam的基线长度，$w,h,l$表示回归的3d_bbox的$width,height,length$，由于所有的keypoint信息都是来自左目，所以$$\left \{\frac{w}{2},\frac{l}{2} \right \}$$的符号根据3d_bbox的角度$\theta$进行相应的改变。

left_2d_bbox的左上点的坐标为$(u_l,v_t)$，则根据3d世界坐标系到3d相机坐标系再到3d图像坐标系之间的透视投影关系得到：
$$
\left[\begin{array}{c}{u_{l}} \\ {v_{t}} \\ {1}\end{array}\right]=K \cdot\left[\begin{array}{c}{x_{\text {cam}}^{t l}} \\ {y_{\text {cam}}^{t l}} \\ {z_{\text {cam}}^{t l}}\end{array}\right] \doteq K \cdot T_{\text {cam}}^{o b j} \cdot\left[\begin{array}{c}{x_{o b j}^{t l}} \\ {y_{o b j}^{t l}} \\ {z_{o b j}^{t l}}\end{array}\right]=\left[\begin{array}{c}{x} \\ {y} \\ {z}\end{array}\right]+\left[\begin{array}{ccc}{\cos \theta} & {0} & {\sin \theta} \\ {0} & {1} & {0} \\ {-\sin \theta} & {0} & {\cos \theta}\end{array}\right] \cdot\left[\begin{array}{c}{-\frac{w}{2}} \\ {-\frac{h}{2}} \\ {-\frac{l}{2}}\end{array}\right]
$$

其中$K$为相机的内参矩阵，$T_{cam}^{obj}$为世界坐标系中的点对应到相机坐标系下的转化矩阵(这是一种刚性变换)，$\text {(.)\_{obj}}$表示在世界坐标系下点的坐标，$\text {(.)\_{cam}}$表示在相机坐标系下的坐标。

left_2d_bbox的右下点的坐标为$(u_r,v_b)$，同理可以得到：
$$
\left[\begin{array}{c}{u_{r}} \\ {v_{b}} \\ {1}\end{array}\right]=K \cdot\left[\begin{array}{c}{x_{c a m}^{t l}} \\ {y_{c a m}^{l l}} \\ {z_{c a m}^{t l}}\end{array}\right] \doteq K \cdot T_{c a m}^{o b j} \cdot\left[\begin{array}{c}{x_{o b j}^{t l}} \\ {y_{o b j}^{t l}} \\ {z_{o b j}^{t l}}\end{array}\right]=\left[\begin{array}{c}{x} \\ {y} \\ {z}\end{array}\right]+\left[\begin{array}{ccc}{\cos \theta} & {0} & {\sin \theta} \\ {0} & {1} & {0} \\ {-\sin \theta} & {0} & {\cos \theta}\end{array}\right] \cdot\left[\begin{array}{c}{\frac{w}{2}} \\ {\frac{h}{2}} \\ {-\frac{l}{2}}\end{array}\right]
$$

同理可以根据透视投影的关系得到right_2d_bbox两个`boundary keypoint`以及`perspective keypoint`，最终整理可以得到论文中的七个等式关系：

$$
\left\{\begin{array}{l}{u_{l}=\left(x-\frac{w}{2} \cos \theta-\frac{l}{2} \sin \theta\right) /\left(z+\frac{w}{2} \sin \theta-\frac{l}{2} \cos \theta\right)} \\ {v_{t}=\left(y-\frac{h}{2}\right) /\left(z+\frac{w}{2} \sin \theta-\frac{l}{2} \cos \theta\right)} \\ {u_{r}=\left(x+\frac{w}{2} \cos \theta+\frac{l}{2} \sin \theta\right) /\left(z-\frac{w}{2} \sin \theta+\frac{l}{2} \cos \theta\right)} \\ {v_{b}=\left(y+\frac{h}{2}\right) /\left(z-\frac{w}{2} \sin \theta+\frac{l}{2} \cos \theta\right)} \\ {u_{l}^{\prime}=\left(x-b-\frac{w}{2} \cos \theta-\frac{l}{2} \sin \theta\right) /\left(z+\frac{w}{2} \sin \theta-\frac{l}{2} \cos \theta\right)} \\ {u_{r}^{\prime}=\left(x-b+\frac{w}{2} \cos \theta+\frac{l}{2} \sin \theta\right) /\left(z-\frac{w}{2} \sin \theta+\frac{l}{2} \cos \theta\right)} \\ {u_{p}=\left(x+\frac{w}{2} \cos \theta-\frac{l}{2} \sin \theta\right) /\left(z-\frac{w}{2} \sin \theta-\frac{l}{2} \cos \theta\right)}\end{array}\right.
$$

其中$b$为双目的基线长(baseline)。以上方程组可用`Gauss-Newton`法求解。

## Dense 3D Box Alignment
这一部分是针对上面得到的3d_bbox进行后处理的流程。作者认为，上述得到的结果都是在`目标检测`层面的信息进行整合加工得到的，而使用更细粒度的语义信息能够很好的改善结果。并且，作者说明了，使用`pixel-level information`的意义不是要获得`pixel-level result`，目标不是对于图像中的每一个点都估计出其3d_depth。这部分的目标参考原文：

> (1) We only solve the disparity of the 3D bounding box center while using the dense object patch, i.e., we use plenty of pixel measurements to solve one single variable. <br>
> (2) To exclude the pixel belonging to the background or other objects, we define a valid RoI as the region is between the left-right boundary keypoints and lies in the bottom halves of the 3D box since the bottom halves of vehicles fits the 3D box more tightly. [参见下图]

<center><img src="/img/in-post/stereo_rcnn/post_process.pdf" width="50%"></center>

> For a pixel located at the normalized coordinate $(u_i,v_i)$ in the valid RoI of the left image, the photometric error can be defined as:

$$
\mathbf{e}_{i}=\left\|I_{l}\left(u_{i}, v_{i}\right)-I_{r}\left(u_{i}-\frac{b}{z+\Delta z_{i}}, v_{i}\right)\right\|
$$

> where we use $I_l,I_r$ to denote the 3-channels RGB vector of left and right image respectively; $\delta z_i = z_i−z$ the depth differences of pixel $i$ with the 3D box center; and $b$ the baseline length. $z$ is the only objective variable we want to solve.

## Result
<center><img src="/img/in-post/stereo_rcnn/result.pdf" width="80%"></center>

<center><img src="/img/in-post/stereo_rcnn/result2.pdf" width="80%"></center>

## Reference

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内