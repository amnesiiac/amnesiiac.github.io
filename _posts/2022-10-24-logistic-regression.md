---
layout: post
title: "Logistic Regression"
author: "twistfatezz"
date: 2022-10-24 20:36
categories: "2022" 
tags:
  - math
---
### # Logistic Distribution
Let X as a continuous random variable, and obedient to Logistic Distribution, then the X's cumulative distribution function is as follows:

$$ F(x)=P(X \leqslant x)=\frac{1}{1+\mathrm{e}^{-(x-\mu) / y}} $$

And X's propability density function is as follows:

$$ f(x)=F^{\prime}(x)=\frac{\mathrm{e}^{-(x-\mu) / \gamma}}{\gamma\left(1+\mathrm{e}^{-(x-\mu) / \gamma}\right)^2} $$

<details>
    <summary><b>F(x) and f(x) with various mu and gamma</b></summary>
    <center><img src="/assets/images/2022/logistic-regression/2.png" width="60%"></center>
    <center><img src="/assets/images/2022/logistic-regression/1.png" width="60%"></center>
</details>

### # Bi-nomial Logistic Regression
Bi-nomial logistic regression is a classification model defined as: $$ P(Y\mid X) $$, which is in the form of parameterized logistic distribution. Bi-momial logistic regression definitions are as follows:

$$ P(Y=1\mid x) = \frac{e^{(w\cdot x+b)}}{1+e^{(w\cdot x+b)}} \qquad P(Y=0\mid x) = \frac{1}{1+e^{(w\cdot x+b)}} $$

Let $$ w=(w^1,w^2...,w^n,b) \quad x=(x^1,x^2...,x^n,1) $$, and simplifications can be made that:

$$ P(Y=1\mid x) = \frac{e^{(w\cdot x)}}{1+e^{(w\cdot x)}} \qquad P(Y=0\mid x) = \frac{1}{1+e^{(w\cdot x)}} $$

Given log odds defined as follows:

$$ logit(p) = log\frac{p}{1-p} $$

Easy to find that Bi-nomial LR model can be interpreted as a linear model:

$$ log\frac{P(Y=1\mid x)}{1-P(Y=1\mid x)} = log\frac{P(Y=1\mid x)}{P(Y=0\mid x)} = w\cdot x $$

### # Bi-nomial LR Model Parameter Estimation
Given training dataset as $$ T=\{(x_1,y_1), (x_2,y_2)...,x(x_N,y_N)\}, \; x_i\in R^n, y_i\in \{0,1\}$$, then Maximum Likelihood Estimation Method can be adopted for model parameter esti-mation.

Let $$ P(Y=1\mid x)=\pi(x), \; P(Y=0\mid x)=1-\pi(x) $$, then the target function is like:

$$ Likelihood\ Function = \prod_{i=1}^{N}[\pi(x_i)]^{y_i} [1-\pi(x_i)]^{1-y_i}$$

To find the Maximum value of Likelihood Function defined above, derive its Log Form as follows:

$$ 
L(w)=Log(Likelihood\ Function) = \sum_{i=1}^N[y_ilog\pi(x_i)]+(1-y_i)log(1-\pi(x_i)) \\
=\sum_{i=1}^N[y_ilog\frac{\pi(x_i)}{1-\pi(x_i)}+log(1-\pi(x_i))] = \sum_{i=1}^N[y_i(w\cdot x)-log(1+e^{(w\cdot x)})]
$$

Then the solution of the optimization problem equals to find the maximum $$L(w)$$, and the esimation value of $$w$$.

The general solution is `Gradient Descent` or `Quasi-Newton Method`.

### # Multi-nomial Logistic Regression
The above Bi-nomial Logistic Regression Model can be generized to get Multi-nomial Model.

$$ 
P(Y=k\mid x)=\frac{e^{(w_k\cdot x)}}{1+\sum_{i=1}^{K-1}e^{(w_k\cdot x)}} \quad ,k=1,2...,K-1 \\
P(Y=K\mid x)=\frac{1}{1+\sum_{k=1}^{K-1}e^{(w_k\cdot x)}}
$$

### # Reference
> Statistical Learning Methods - LiHang

### # Keyword Desc
> cumulative distribution function: CDF <br>
> propability density function: PDF

