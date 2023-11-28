---
layout: post
title: "Homogeneous coordinates"
subtitle: '齐次坐标简单解析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-02-12 22:01 
lang: ch 
catalog: true 
categories: documentation
tags:
  - imaging principle 
  - Time 2020
---
## Abstract
本文主要对于计算机视觉中齐次坐标的概念进行介绍，方便进一步研究计算机视觉中的[相机参数矩阵矫正](/documentation/2020/02/15/post-camera-matrix/)、[透视投影变换](/documentation/2019/11/24/post-perspective-projection/)等。本文章主要对于[Homogeneous Coordinates](http://www.songho.ca/math/homogeneous/homogeneous.html)博客中的内容进行翻译简析。

## Problem 
通常情况下，如下图所示，在欧式几何空间，同一平面的两条平行线不能相交。下图中展示了二维和三维欧式空间。
<center><img src="/img/in-post/homogeneous_coordinates/euclidean.pdf" width="60%"></center>

然而，广泛用于相机成像的透视投影学中，现实世界中的平行线将会相交于一点，称之为`消隐点`或`vanishing point`。
<center><img src="/img/in-post/homogeneous_coordinates/perspective.pdf" width="60%"></center>

欧式几何坐标系，或者称为笛卡尔坐标系对于2d、3d等建立坐标定量分析非常适用，但是它们不能够正确处理透视几何问题。换一种说法，欧式几何是透视几何的一个子集合。在欧式几何中，两条平行线通常被认为在无穷远点相交，在二维欧式空间下，此交点可表示为$(-\infty,+\infty)$，这个坐标在欧式空间中无意义。因此，我们需要换一种坐标的表示方式，来解决这个问题。

## Solution
`Homogeneous coordinates`被August Ferdinand Möbius所提出以解决定量分析透视投影空间下的图形几何学问题。齐次坐标表示法是一种使用$N+1$维坐标来表示$N$维空间的一种方法。<br>
假设欧式坐标为$(x,y)$，则我们增加一个变量$w$来将二维欧式空间中的点`E`:$(x,y)$映射到齐次坐标空间中的点`H`:$(X,Y)$，并用$(x,y,w)$对`H`进行表示。其映射的规则如下：$X=x/w$和$Y=y/w$。

假设欧式空间的点坐标为$(1,2)$，在齐次坐标系下可以表示成$(1,2,1)$或者$(2,4,2)$，表示的方式不唯一。这样一来，对于欧式无穷远点$(-\infty,+\infty)$就可以用齐次坐标表示成$(1,2,0)$或者$(2,4,0)$。

## Why Homogeneous?
根据上面的描述，欧式坐标和齐次坐标可以由下面的式子进行转化。

$$
Homogeneous \quad \quad Cartesian \\
\quad (x, y, w) \quad \Leftrightarrow \quad\left(\frac{x}{w}, \frac{y}{w}\right)
$$

以齐次坐标空间中的点$(1,2,3)$为例，从齐次坐标空间转化到欧式空间(笛卡尔系)中去。

$$
\begin{array}{l}
\scriptsize{Homogeneous} \ \ \ \ \ \scriptsize{Cartesian} \\
{(1,2,3) \ \ \ \Rightarrow \ \left(\frac{1}{3}, \frac{2}{3}\right)} \\
{(2,4,6) \ \ \ \Rightarrow \ \left(\frac{2}{6}, \frac{4}{6}\right)=\left(\frac{1}{3}, \frac{2}{3}\right)} \\
{(4,8,12) \ \ \Rightarrow \ \left(\frac{4}{12}, \frac{8}{12}\right)=\left(\frac{1}{3}, \frac{2}{3}\right)} \\
\quad \ \ \vdots \quad \quad \quad \quad \quad \quad \vdots \\
(1a,2a,3a) \Rightarrow\left(\frac{1a}{3a},\frac{2a}{3a}\right)=\left(\frac{1}{3},\frac{2}{3}\right)
\end{array}
$$

根据上式，齐次坐标系中的点$(1,2,3)$、$(2,4,6)$、$(4,8,12)$和同一个欧式空间中的点$(1/3, 2/3)$相互关联，这种情况下，我们称之为$$They\ are\ homogeneous$$。

## A proof
这一部分主要用于证明**Two parallel lines can intersect**。<br>
考虑到欧式空间中的两条平行直线：

$$
\left\{\begin{array}{l}{A x+B y+C=0} \\ {A x+B y+D=0}\end{array}\right.
$$

我们很容易知道，如果$C \neq D$，那么上式无解，如果$C=D$，两条直线为同一条线。

将欧式坐标和齐次坐标之间的转化关系$X=x/w$以及$Y=y/w$带入得到：

$$
\begin{aligned}
\left\{\begin{array}{l}  {A \frac{x}{w}+B\frac{y}{w}+C=0} \\ A {\frac{x}{w}+B\frac{y}{w}+D=0} \end{array}\right. & \Rightarrow 
\left\{\begin{array}{l} {A x+B y+Cw=0} \\ {A x+B y+Dw=0} \end{array}\right.
\end{aligned}
$$

此时，平行直线在齐次坐标空间中有解：$(x,y,w)=(x,y,0)$，对应了欧式空间中的无穷远点。

> `补充:` Homogeneous coordinates are very useful and fundamental concept in computer graphics, such as projecting a 3D scene onto a 2D plane.

## Reference
> 翻译自http://www.songho.ca/math/homogeneous/homogeneous.html

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内