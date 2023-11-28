---
layout: post
title: "Perspective Projection"
subtitle: 'Mathematical Connection between obj&cam&pic'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-11-24 10:11 
lang: ch 
catalog: true 
categories: documentation
tags:
  - imaging principle 
  - Time 2019
---
## Basic concepts 
**[三大坐标系]**
投射投影变换涉及到三个坐标系，主要包括世界坐标系、摄像机坐标系、图像坐标系。<br>
**(世界坐标系)**是真实世界中目标物体位置的参考坐标系。除了无穷远，世界坐标可以根据运算方便与否自由放置。在双目视觉中主要有三个用途：1）标定真实世界目标物的位置. 2)作为双目相机的参考系：给出双目相机相对于世界坐标系的位置，求解两个相机之间的相对关系. 3)作为双目3d重建好的物体的"容器"，存放重建之后的物体的三维坐标. <br>
**(摄像机坐标系)**摄像机站在自己角度上衡量的物体的坐标系。摄像机坐标系的原点在摄像机的光心上，$z$轴与摄像机光轴平行。世界坐标系下的物体需先经历`刚体变化`转到摄像机坐标系，然后在和图像坐标系发生关系。 <br>
**(图像坐标系)**图像平面的中心为坐标原点，为了描述成像过程中物体从相机坐标系到图像坐标系的投影透射关系而引入，方便进一步得到像素坐标系下的坐标。图像坐标系是用物理单位(如mm)表示像素在图像中的位置。 <br>
**(像素坐标系)**像素坐标系$(u,v)$，以图像平面的左上角顶点为原点，为了描述物体成像后的像点在数字图像上的坐标而引入，是我们真正从相机内读取到的信息所在的坐标系。像素坐标系就是以像素为单位的图像坐标系。可以将图像和像素坐标系看成一组坐标系。

其中可以将图像坐标系和像素坐标系看成一个坐标系，称为``三大坐标系''，也可以将其分开考虑，称为四大坐标系。

## Transformation 
**[三大坐标系之间的转换]**
三大坐标系一般指的是`世界坐标系`，`摄像机坐标系`以及`图像坐标系`。他们三者之间的关系是`世界坐标系`经过刚体变化可以转化成`摄像机坐标系`，`摄像机坐标系`又可以通过透视投影变化转化成`图像坐标系`。

### img to pixel
**(从图像坐标系到像素坐标系)**图像坐标系是用来衡量图像在真实世界中，各个像素之间的关系，因此，图像坐标系中的像素使用真实世界中的度量指标如mm对于每个像素之间的位置关系进行衡量，主要用来研究物体-相机-图像三者之间的透视投影关系。像素坐标系用来衡量当前相机成像的图像内部元素的位置`相对位置关系`的坐标系。两个坐标系之间的关系如下图：

<center><img src="/img/in-post/perspective_projection/transform1.pdf" width="40%"></center>

上面图像中展示了两种坐标系的大概关系，其中`Oxy`图像坐标系中的点以物理单位进行描述，`Ouv`像素坐标系中的点以''行数和列数''进行描述。通过数学公式对于上述两个坐标系中的点进行定量描述如下：

$$
u=\frac{x}{dx}+u_0  \quad v=\frac{y}{dy}+v_0
$$

其中，假设图像坐标系中的点以毫米(mm)进行描述，那么$dx$的单位则为毫米/像素，所以$x/dx$的单位即为像素(像素坐标系)。$u_0$以及$v_0$表示两个坐标系之间的以像素为单位的平移关系(可以通过两个坐标系的原点之间的平移关系获得)。进一步地，可以将上述关系通过矩阵关系表示成下面的形式：

$$
\left[\begin{array}{l}
{u} \\
{v} \\
{1}
\end{array}\right]=\left[\begin{array}{ccc}
{1 / d x} & {0} & {u_{0}} \\
{0} & {1 / d y} & {v_{0}} \\
{0} & {0} & {1}
\end{array}\right]\left[\begin{array}{l}
{x} \\
{y} \\
{1}
\end{array}\right]
$$

在上述公式中使用了齐次坐标，那么为什么需要加一维变成齐次坐标的形式呢?对于像素坐标点来说，定义$(u,v,1)$表示平面中$(u,v)$的有限远处的点，定义$(u,v,0)$表示无穷远点。增加一维坐标(使用齐次坐标)的优点是:1)定义了无穷远点(也可称之为消隐点(vanishing point))的坐标表示方式。2)采用这种形式表达使得整个计算方式更加规整。3)齐次坐标具有一个重要性质：伸缩不变形，即设齐次坐标$M$，则有$\alpha M=M$。更多关于齐次坐标的内容，可以参考[这个博客](http://www.songho.ca/math/homogeneous/homogeneous.html)，也可以参考我对于这个博客的[翻译整理](/documentation/2020/02/03/post-homogeneous-coordinates/)。

### world to camera
**(从世界坐标系到摄像机坐标系)**首先需要介绍刚体变换:空间中，物体在不发生形变的情况下，对于该物体只做`旋转`以及`平移`运动的变换称之为刚体变换。

<center><img src="/img/in-post/perspective_projection/transform2.pdf" width="40%"></center>

上述刚体变换的数据表达式可以表达成如下形式：

$$
\left[\begin{array}{c}
{X_{C}} \\
{Y_{C}} \\
{Z_{C}}
\end{array}\right]=\left[\begin{array}{ccc}
{r_{00}} & {r_{01}} & {r_{02}} \\
{r_{10}} & {r_{11}} & {r_{12}} \\
{r_{20}} & {r_{21}} & {r_{22}}
\end{array}\right]\left[\begin{array}{c}
{X_{W}} \\
{Y_{W}} \\
{Z_{W}}
\end{array}\right]+\left[\begin{array}{c}
{T_{X}} \\
{T_{Y}} \\
{T_{Z}}
\end{array}\right]
$$

公式中的坐标转化矩阵是一个`旋转矩阵的形式`， 后面的向量是一个平移向量，两者的结合构成了`刚体变换`。将三维刚性变换的表达式转变成齐次坐标表达的形式，可以写成如下形式：

$$
\left[\begin{array}{c}
{X_{C}} \\
{Y_{C}} \\
{Z_{C}} \\
{1}
\end{array}\right]=\left[\begin{array}{cc}
{R} & {t} \\
{0_{3}^{T}} & {1}
\end{array}\right]\left[\begin{array}{c}
{X_{W}} \\
{Y_{W}} \\
{Z_{W}} \\
{1}
\end{array}\right]=\left[\begin{array}{cccc}
{r_{1}} & {r_{2}} & {r_{3}} & {t}
\end{array}\right]\left[\begin{array}{c}
{X_{W}} \\
{Y_{W}} \\
{0} \\
{1}
\end{array}\right]=\left[\begin{array}{ccc}
{r_{1}} & {r_{2}} & {t}
\end{array}\right]\left[\begin{array}{c}
{X_{W}} \\
{Y_{W}} \\
{1}
\end{array}\right]
$$

同刚性变换的公式表达，齐次变换的公式表达中的$R$矩阵是$3\times 3$正交单位矩阵(即旋转矩阵)，$t$表示平移向量，其中$R$矩阵和$T$矩阵和摄像机无关，称之为摄像机的`外参数`。在后面的坐标系关系推导的过程中，不考虑两个坐标系之间旋转的情况，方便后续计算的方便表达。

### camera to img
**(从摄像机坐标系到图像坐标系)**两个坐标系之间是通过`透视投影关系`进行联系的，下面简单介绍`透视投影变换`：用中心投影法将形体投射到投影面上，从而获得的一种较为接近视觉效果的单面投影图。这种投影方式符合正常眼睛观察事物的认知，即相对视点近大远小的原理，而且不平行于成像平面的平行线会相交与消隐点(vanish point)。下面以针孔相机成像模型为例，简单介绍投影变换原理。下图中，$\pi$平面称之为摄像机的像平面，点$O_c$称之为摄像机中心(光心)，$f$是摄像机的焦距，$O_c$作为一端且垂直于像平面$\pi$的涉嫌称之为光轴或者主轴，主轴和像平面$\pi$的交点$p$称为摄像机主点。

<center><img src="/img/in-post/perspective_projection/transform3.pdf" width="60%"></center>

图像中，$O_c$-$x_c,y_c$为图像坐标系，$O_x$-$x_c,y_c,z_c$为摄像机坐标系。空间中的点$X_c$在摄像机坐标系中的齐次坐标为$X_c=(x_c,y_c,z_c,1)^{T}$，点$X_c$在像平面中的投影点$m$在图像坐标系中的齐次坐标为$m=(x,y,1)^T$，因此从摄像机坐标系到图像坐标系之间的关系透视投影变换可以通过数学上的三角相似关系进行表达，利用上图中三角形$O_c,x,y$和三角形$O_c,x_c,y_c$之间的相似关系，可以得到：

$$
\left\{\begin{array}{l}
{\boldsymbol{x}=\frac{f x_{c}}{z_{c}}} \\
{\boldsymbol{y}=\frac{f y_{c}}{z_{c}}}
\end{array}\right.
$$

将上述关系通过矩阵形式进行表达，可以表示为：

$$
z_{c}\left[\begin{array}{l}
{x} \\
{y} \\
{1}
\end{array}\right]=\left[\begin{array}{llll}
{f} & {0} & {0} & {0} \\
{0} & {f} & {0} & {0} \\
{0} & {0} & {1} & {0}
\end{array}\right]\left[\begin{array}{l}
{x_{c}} \\
{y_{c}} \\
{z_{c}} \\
{1}
\end{array}\right]=\left[\begin{array}{lll}
{f} & {0} & {0} \\
{0} & {f} & {0} \\
{0} & {0} & {1}
\end{array}\right]\left[\begin{array}{llll}
{1} & {0} & {0} & {0} \\
{0} & {1} & {0} & {0} \\
{0} & {0} & {1} & {0}
\end{array}\right]\left[\begin{array}{l}
{x_{c}} \\
{y_{c}} \\
{z_{c}} \\
{1}
\end{array}\right]
$$

注意，其中由于是齐次坐标的表示形式，具有伸缩不变性，因此$Z_c\left[x,y,1 \right]$和$\left[x,y,1 \right]$表示同一个点。

## One-step transform  
**(从图像坐标系到摄像机坐标系到世界坐标系)**现在一步到位，将三大坐标系之间的转化通过齐次坐标表达形式，进行转化。下面的公式中$z_c\left[u,v,1 \right]$是图像坐标系中的点的齐次坐标表达形式，$\left[x_c,y_c,z_c,1 \right]$表示摄像机坐标系中的点的齐次坐标表达形式。

$$
z_{c}\left[\begin{array}{c}
{u} \\
{v} \\
{1}
\end{array}\right]=z_{c}\left[\begin{array}{ccc}
{\frac{1}{d_{x}}} & {0} & {u_{0}} \\
{0} & {\frac{1}{d_{y}}} & {v_{0}} \\
{0} & {0} & {1}
\end{array}\right]\left[\begin{array}{l}
{x} \\
{y} \\
{1}
\end{array}\right]=\left[\begin{array}{ccc}
{\frac{1}{d_{x}}} & {0} & {u_{0}} \\
{0} & {\frac{1}{d_{y}}} & {v_{0}} \\
{0} & {0} & {1}
\end{array}\right]\left[\begin{array}{cccc}
{f} & {0} & {0} & {0} \\
{0} & {f} & {0} & {0} \\
{0} & {0} & {1} & {0}
\end{array}\right]\left[\begin{array}{c}
{x_{c}} \\
{y_{c}} \\
{z_{c}} \\
{1}
\end{array}\right]
$$

$$
=\left[\begin{array}{ccc}
{\frac{1}{d_{x}}} & {0} & {u_{0}} \\
{0} & {\frac{1}{d_{y}}} & {v_{0}} \\
{0} & {0} & {1}
\end{array}\right]\left[\begin{array}{cccc}
{f} & {0} & {0} & {0} \\
{0} & {f} & {0} & {0} \\
{0} & {0} & {1} & {0}
\end{array}\right]\left[\begin{array}{cccc}
{r_{00}} & {r_{01}} & {r_{02}} & {T_{X}} \\
{r_{10}} & {r_{11}} & {r_{12}} & {T_{Y}} \\
{r_{20}} & {r_{21}} & {r_{22}} & {T_{Z}} \\
{0} & {0} & {0} & {1}
\end{array}\right]\left[\begin{array}{c}
{x_{w}} \\
{y_{w}} \\
{z_{w}} \\
{1}
\end{array}\right]
$$

其中可以定义世界坐标系中的点转换到图像坐标系中的点的转换矩阵$P$如下：

$$
P=\left[\begin{array}{cccc}
{P_{00}} & {P_{01}} & {P_{02}} & {P_{03}} \\
{P_{10}} & {P_{11}} & {P_{12}} & {P_{13}} \\
{P_{20}} & {P_{21}} & {P_{22}} & {P_{23}} \\
\end{array}\right]=
\left[\begin{array}{ccc}
{\frac{1}{d_{x}}} & {0} & {u_{0}} \\
{0} & {\frac{1}{d_{y}}} & {v_{0}} \\
{0} & {0} & {1}
\end{array}\right]\left[\begin{array}{cccc}
{f} & {0} & {0} & {0} \\
{0} & {f} & {0} & {0} \\
{0} & {0} & {1} & {0}
\end{array}\right]\left[\begin{array}{cccc}
{r_{00}} & {r_{01}} & {r_{02}} & {T_{X}} \\
{r_{10}} & {r_{11}} & {r_{12}} & {T_{Y}} \\
{r_{20}} & {r_{21}} & {r_{22}} & {T_{Z}} \\
{0} & {0} & {0} & {1}
\end{array}\right]
$$

最后总结不考虑畸变的情况下，三大坐标系之间的转换关系：
<center><img src="/img/in-post/perspective_projection/transform4.pdf" width="80%"></center>

## Reference
> https://www.cnblogs.com/zyly/p/9366080.html#_label1

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内