---
layout: post
title: "logistic regression"
author: "melon"
date: 2022-10-24 20:36
categories: "2022" 
tags:
  - math
---

let X as a continuous random variable, and obedient to logistic distribution,
then X's cumulative distribution function is as follows:

$$ F(x)=P(X \leqslant x)=\frac{1}{1+\mathrm{e}^{-(x-\mu)/\gamma}} $$

and X's propability density function is as follows:

$$ f(x)=F^{\prime}(x)=\frac{\mathrm{e}^{-(x-\mu) / \gamma}}{\gamma\left(1+\mathrm{e}^{-(x-\mu) / \gamma}\right)^2} $$

CDF and PDF function morphology is as:

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/lr.pdf" width="600"/>

<hr>

### # bi-nomial logistic regression
bi-nomial logistic regression is a classification model defined as: $$ P(Y\mid X) $$, which is in the form of parameterized logistic distribution.
bi-nomial logistic regression definitions are as follows:

$$ P(Y=1\mid x) = \frac{e^{(w\cdot x+b)}}{1+e^{(w\cdot x+b)}} \qquad P(Y=0\mid x) = \frac{1}{1+e^{(w\cdot x+b)}} $$

let $$ w=(w^1,w^2...,w^n,b) \quad x=(x^1,x^2...,x^n,1) $$, and simplifications can be made that:

$$ P(Y=1\mid x) = \frac{e^{(w\cdot x)}}{1+e^{(w\cdot x)}} \qquad P(Y=0\mid x) = \frac{1}{1+e^{(w\cdot x)}} $$

given logarithmic odds defined as follows:

$$ logit(p) = log\frac{p}{1-p} $$

it's easy to find: bi-nomial LR model can be interpreted as a linear model:

$$ log\frac{P(Y=1\mid x)}{1-P(Y=1\mid x)} = log\frac{P(Y=1\mid x)}{P(Y=0\mid x)} = w\cdot x $$

<hr>

### # bi-nomial logistic regression model parameter estimation
given training dataset as $$ T=\{(x_1,y_1), (x_2,y_2)...,x(x_N,y_N)\}, \; x_i\in R^n, y_i\in \{0,1\}$$,
then maximum likelihood estimation method can be adopted for model parameter estimation.

let $$ P(Y=1\mid x)=\pi(x), \; P(Y=0\mid x)=1-\pi(x) $$, then the target function is like:

$$ Likelihood\ Function = \prod_{i=1}^{N}[\pi(x_i)]^{y_i} [1-\pi(x_i)]^{1-y_i}$$

to find the maximum value of likelihood function defined above, derive its log form:

$$
L(w)=Log(Likelihood\ Function) = \sum_{i=1}^N[y_ilog\pi(x_i)]+(1-y_i)log(1-\pi(x_i)) \\
=\sum_{i=1}^N[y_ilog\frac{\pi(x_i)}{1-\pi(x_i)}+log(1-\pi(x_i))] = \sum_{i=1}^N[y_i(w\cdot x)-log(1+e^{(w\cdot x)})]
$$

the solution to the optimization problem equals to find the maximum $$L(w)$$,
and corresponding parameter $$w$$, which can resort to gradient descent or quasi-newton for help.

<hr>

### # multi-nomial logistic regression
the above bi-nomial logistic regression model can be generized to get multi-nomial model.

$$ 
P(Y=k\mid x)=\frac{e^{(w_k\cdot x)}}{1+\sum_{i=1}^{K-1}e^{(w_k\cdot x)}} \quad \quad k=1,2...,K-1 \\
P(Y=K\mid x)=\frac{1}{1+\sum_{k=1}^{K-1}e^{(w_k\cdot x)}}
$$

the parameter estimation method for bi-nomial can also be generalized to multi-nomial param estimation.

<hr>

### # reference
statistical learning methods - LiHang

<hr>

### # keyword desc
cumulative distribution function: CDF  
propability density function: PDF

