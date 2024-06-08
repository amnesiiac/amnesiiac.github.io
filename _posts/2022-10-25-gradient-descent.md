---
layout: post
title: "gradient descent method"
author: "melon"
date: 2022-10-25 22:22
categories: "2022" 
tags:
  - math
---

in mathematics, gradient descent (also often called steepest descent) is a first-order
iterative optimization algorithm for finding a local minimum of a differentiable function.

the idea is to take repeated steps in the opposite direction of the gradient
(or approximate gradient) of the function at the current point,
because this is the direction of steepest descent.

conversely, stepping in the direction of the gradient will lead to a local maximum
of that function; the procedure is then known as gradient ascent.

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/gd.pdf" width="560"/>

<hr>

### # comprehension through metaphors (2d)
the basic intuition behind gradient descent can be illustrated by a hypothetical scenario.

a person is stuck in the mountains and is trying to get down (trying to find the global minimum).
there is heavy fog such that visibility is extremely low.
therefore, the path down the mountain is not visible, so they must use local information to find the minimum.

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/gd2.pdf" width="360"/>

a algorithm named gradient descent can be used to solve this,
which involves looking at the steepness of the hill at their current position,
then proceeding in the direction with the steepest descent (downhill).

if they were trying to find the top of the mountain (i.e., the maximum),
then they would proceed in the direction of steepest ascent (uphill).

using this method, they would eventually find their way down the mountain or
possibly get stuck in some hole (local minimum or saddle point), e.g. a mountain lake.

<hr>

### # choose step size & descent direction
using a step size that is too small would slow convergence, and a too large step would lead to divergence,
finding a good step is an important practical problem.

moreover, the descent direction also matters in this convergence problem.
wolfe conditions, through which the finding of local minima is guaranteed.

todo: get more intro about gradient decent from the view of math and code.

<hr>

### # reference
https://en.wikipedia.org/wiki/Gradient_descent  
https://en.wikipedia.org/wiki/Wolfe_conditions
