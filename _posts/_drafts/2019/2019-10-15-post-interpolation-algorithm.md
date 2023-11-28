---
layout: post
title: "Interpolation Algorithm"
subtitle: '原理简要解析'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2019-10-15 14:32 
lang: ch 
catalog: true 
categories: documentation
tags:
  - imaging principle 
  - Time 2019
---

## Introduction
这篇文章旨在记录单线性插值和双线性插值算法，作为备忘。线性插值在图像处理中是基础的方法，需要掌握。在图像上采样，图像`浮点位置元素值推断`中(如[mask-rcnn](/documentation/2019/10/16/post-mask-rcnn/)中的`roi-align`)，这种插值的方式被广泛的应用。

## Linear Interpolation
已知数据$(x0,y0)$与$(x1,y1)$，计算$[x0, x1]$区间内某一位置$x$在直线上的$y$值。用图像表达这个问题：

<center><img src="/img/in-post/interpolation/linear_interpolation.png" width="40%"></center>

使用3点共线的条件即可：

$$
\frac{y-y0}{x-x0}=\frac{y1-y0}{x1-x0} \ 即\ y=\frac{x1-x}{x1-x0}y0 + \frac{x-x0}{x1-x0}y1 
$$

## Bilinear Interpolation
双线性插值是有两个变量的插值函数的线性插值扩展，其核心思想是在两个方向分别进行一次线性插值。<br>
双线性插值解决的问题是：已知数据$Q11,Q12,Q21,Q22$四个位置(坐标)及其数值(函数值)$f(Q11)$,$f(Q12)$,$f(Q21)$,$f(Q22)$，计算位于上面四个位置框出的区域中给定位置$(x,y)$的数值$f(P)$。用图像表达这个问题：

<center><img src="/img/in-post/interpolation/bilinear_interpolation.png" width="40%"></center>

分别对于$x$方向以及$y$方向应用单线性插值即可：<br>
对于$x$方向应用两次线性插值，分别得到$R1,R2$位置对应的数值$f(R1),f(R2)$：

$$
f\left(R_{1}\right) \approx \frac{x_{2}-x}{x_{2}-x_{1}} f\left(Q_{11}\right)+\frac{x-x_{1}}{x_{2}-x_{1}} f\left(Q_{21}\right) \quad \text {where} \quad R_{1}=\left(x, y_{1}\right) 
$$

$$
f\left(R_{2}\right) \approx \frac{x_{2}-x}{x_{2}-x_{1}} f\left(Q_{12}\right)+\frac{x-x_{1}}{x_{2}-x_{1}} f\left(Q_{22}\right) \quad \text {where} \quad R_{2}=\left(x, y_{2}\right)
$$

对于$y$方向，对于$R1,R2$在运用一次单线性插值得到：

$$
f(P) \approx \frac{y_{2}-y}{y_{2}-y_{1}} f\left(R_{1}\right)+\frac{y-y_{1}}{y_{2}-y_{1}} f\left(R_{2}\right)
$$

最后，将$x$方向的公式代入$y$方向的公式，得到：

$$
f(x, y) \approx \frac{f\left(Q_{11}\right)}{\left(x_{2}-x_{1}\right)\left(y_{2}-y_{1}\right)}\left(x_{2}-x\right)\left(y_{2}-y\right)+\frac{f\left(Q_{21}\right)}{\left(x_{2}-x_{1}\right)\left(y_{2}-y_{1}\right)}\left(x+x_{1}\right)\left(y_{2}-y\right) 
$$

$$
+\frac{f\left(Q_{12}\right)}{\left(x_{2}-x_{1}\right)\left(y_{2}-y_{1}\right)}\left(x_{2}-x\right)\left(y-y_{1}\right)+\frac{f\left(Q_{22}\right)}{\left(x_{2}-x_{1}\right)\left(y_{2}-y_{1}\right)}\left(x-x_{1}\right)\left(y-y_{1}\right)
$$

## Reference

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内