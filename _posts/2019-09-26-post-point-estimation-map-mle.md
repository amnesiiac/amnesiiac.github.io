---
layout: post
title: "MLE vs MAP (math, point estimation)"
author: "melon"
date: 2019-09-26 10:36
categories: "2019"
tags:
  - math
---

MLE(maximum likelihood estimation) and MAP(maximum a posteriori) are all point estimations,
which are used for estimating values of variables in probability distributions & graphical models.

MLE and MAP are rough the same, for both are used for computing a single estimate, rather than
getting the full distribution.

while, the MLE is view from Frequency, MAP is view from Bayesian.

<hr>

### # coin test: difference between MLE & MAP
a coin toss game repeat 5 times, result in 1 head 4 tails.

(1) in the view of MLE: each time the result of a coin toss is a random variable,
and the PDF of this game var should conform to 0-1 distribution.  
hence, MLE view is to directly use the observed result as ref, then compute the param p
in 0-1 distribution, which can reproduce the ref result to the max extend.

(2) in the view of MAP: the coin is only tossed 5 times, which is not fully convincible,
therefore, we want to gain some prior knownledge: whether the coin is balanced & symmetrical\...
investigation shows: the coin is made by a mint, ideally the coin is perfect, and param p = 0.5,
however, we want to believe: the coin can be perfect on 1000 factory tests, combining
prior knowledge & local test, the param p can be computed as:

$$ (0.2*5 + 0.5*1000) / (1000+5) = 0.4985 $$

if we do not trust the coin mint so much, only believe the coin be perfect on 5 factory tests,
the param p can be computed as:

$$ (0.2*5 + 0.5*5) / (5+5) = 0.35 $$

<hr>

### # maximum likelihood estimation (MLE)
let $$P(X\mid  \theta)$$ denotes maximum likelihood function, the parameter $$\theta$$ can
be resolved by:

$$\theta_{MLE}=\underset{\theta}{\arg \max } P(X\mid  \theta)=\underset{\theta}{\arg \max } \prod_{i} P\left(x_{i}\mid \theta\right)$$

however, for many samples(xi), $$P(x_i\mid \theta)$$ is much approach to 0, which makes above
formular unable to compute (due to computation underflow).

mapping the formular into logarithmic space for computation:

$$\theta_{MLE}=\underset{\theta}{\arg\max} \log P(X\mid \theta)=\underset{\theta}{\arg\max} \log \prod_{i} P\left(x_{i}\mid \theta\right)=\underset{\theta}{\arg\max} \sum_{i} \log P\left(x_{i}\mid \theta\right)$$

the optimization problem above can be solved by gradient descentdant method or other variants.

<hr>

### # maximum a posterior (MAP)
for maximum posterior, the likelihood function and prior distribution of random variable
both need to take into consideration. according to Beyesian's law:

$$P(\theta\mid X)=\frac{P(X\mid \theta) P(\theta)}{P(X)} \propto P(X\mid \theta) P(\theta)$$

in which the prior distribution can be seen as a constant, and posterior probability is only
relative to MLE + param distribution.

$$\theta_{MAP}=\underset{\theta}{\arg\max} P(X\mid \theta) P(\theta)\\
=\underset{\theta}{\arg\max} \log P(X\mid \theta)+\log P(\theta)\\
=\underset{\theta}{\arg\max} \log \prod_{i} P\left(x_{i}\mid \theta\right)+\log P(\theta)\\
=\underset{\theta}{\arg \max } \sum_{i} \log P\left(x_{i}\mid \theta\right)+\log P(\theta)$$

comparing MAP with MLE: the only difference lies in $$logP(\theta)$$, if $$logP(\theta)$$
is not relative to $$\theta$$, e.g. uniform distribution, then MAP is the same as MLE.

in the context of coin test: if probability of coin head or tail is nothing to do with coin
quality, then there's no need to do factory test before release.

<hr>

### # reference
https://wiseodd.github.io/techblog/2017/01/01/mle-vs-map/  
https://en.wikipedia.org/wiki/Maximum_likelihood_estimation

cumulative distribution function: CDF  
propability density function: PDF
