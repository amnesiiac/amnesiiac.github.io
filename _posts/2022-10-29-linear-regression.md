---
layout: post
title: "linear regression"
author: "melon"
date: 2022-10-29 09:52
categories: "2022" 
tags:
  - math
---

### # introduction
in statistics, linear regression is a linear approach for modelling the relationship
between a scalar response and one or more explanatory variables (dependent and independent
variables). 

<hr>

### # variants
the case of one explanatory variable is called simple linear regression:

$$y=f(x)$$

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/lr2.pdf" width="300"/>

for more than one explanatory variables, the process is called multiple linear regression:

$$y=f(x_1,x_2...,x_n)$$

for more then one response variables, the process is called multivariate linear regression:

$$y_i=f_i(x)$$

finally, we have multivariate multiple regression:

$$y_i=f_i(x_1,x_2...,x_n)$$

the linear expression in matrix form:

$$ Y=XW+\epsilon $$

$$
Y=\left(\begin{array}{c}
y_1 \\ y_2 \\ \vdots \\ y_n
\end{array}\right)
\ 
X = 
\left(\begin{array}{c}
\mathbf{x}_1^{\top} \\
\mathbf{x}_2^{\top} \\
\vdots \\
\mathbf{x}_n^{\top}
\end{array}\right) 
=
\left(\begin{array}{cccc}
1 & x_{11} & \cdots & x_{1 p} \\
1 & x_{21} & \cdots & x_{2 p} \\
\vdots & \vdots & \ddots & \vdots \\
1 & x_{n 1} & \cdots & x_{n p}
\end{array}\right)
\ 
W = 
\left(\begin{array}{c}
w_o \\ w_1 \\ \vdots \\ w_p
\end{array}\right)
\ 
\epsilon =
\left(\begin{array}{c}
\epsilon_o \\ \epsilon_1 \\ \vdots \\ \epsilon_p
\end{array}\right)
$$

where Y is a matrix with series of multivariate measurements,
X is a matrix of observations on independent variables,
W is a matrix contains parameters to be estimated,
and epsilon is a matrix contains errors (noise).

<hr>

### # math solution of 2d linear regression
given linear regression model as following formula:

$$ y = H_w(x) = w_1 + w_2 \times x $$

to find the best formula paramters which fits the given sample data points
$(x_1,y_1), (x_2,y_2)...,(x_n,y_n)$ best, given the loss function (MSE) below,
then the best fitting line problem is equivalent to:

$$ Minimize\ J(w_1,w_2) = \frac{1}{n}\sum_{i=1}^n(pred_i-y_i)^2 $$

there is no harm in adding a multiplier factor:

$$ Minimize\ J(w_1,w_2) = \frac{1}{2n}\sum_{i=1}^n(pred_i-y_i)^2 $$

the equivalent form of the formula above is as follows:

$$ Minimize\ J(w_1, w_2) = \frac{1}{2n}\sum_{i=1}^n(H_w(x_i)-y_i)^2 $$

by gradient descent method, the parameter updating is as follows:

$$ w_t = w_t - \alpha\frac{\partial}{\partial w_t}J(w_1,w_2) $$

$$
Where \quad \frac{\partial}{\partial w_t}J_w = \frac{\partial}{\partial w_t}\frac{1}{2n}\sum_{i=1}^n[H_w(x_i)-y_i]^2 = \frac{1}{n}(H_w(x_i)-y)x_i
$$

$$ Finally \quad w_t = w_t - \frac{\alpha}{n}\sum_{i=1}^n[(H_w(x_i)-y_i)x_i] $$

<hr>

### # example
todo

<hr>

### # reference
https://en.wikipedia.org/wiki/Linear_regression  
