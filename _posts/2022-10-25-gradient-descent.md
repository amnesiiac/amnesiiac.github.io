---
layout: post
title: "Gradient Descent Method"
author: "twistfatezz"
date: 2022-10-25 22:22
categories: "2022" 
tags:
  - math
---
### # Introduction
In mathematics, gradient descent (also often called steepest descent) is a first-order iterative optimization algorithm for finding a local minimum of a differentiable function. 

The idea is to take repeated steps in the opposite direction of the gradient (or approximate gradient) of the function at the current point, because this is the direction of steepest descent. Conversely, stepping in the direction of the gradient will lead to a local maximum of that function; the procedure is then known as gradient ascent.

<details>
    <summary><b>Gradient Descent in 1 Dimension</b></summary>
    <center><img src="/assets/images/2022/gradient-descent/1.png" width="100%"></center>
</details>

### # Comprehension Through Metaphors (2d)
The basic intuition behind gradient descent can be illustrated by a hypothetical scenario. 

A person is stuck in the mountains and is trying to get down (trying to find the global minimum). There is heavy fog such that visibility is extremely low. Therefore, the path down the mountain is not visible, so they must use local information to find the minimum. 

<details>
    <summary><b>Fog in the Mountain</b></summary>
    <center><img src="/assets/images/2022/gradient-descent/fog.jpg" width="60%"></center>
</details>

They can use the method of gradient descent, which involves looking at the steepness of the hill at their current position, then proceeding in the direction with the steepest descent (downhill). 

If they were trying to find the top of the mountain (i.e., the maximum), then they would proceed in the direction of steepest ascent (uphill). 

Using this method, they would eventually find their way down the mountain or possibly get stuck in some hole (local minimum or saddle point), like a mountain lake. 

### # Choose Step Size & Descent Diretion (Convergence)
Using a step size that is too small would slow convergence, and a too large step would lead to divergence, finding a good step is an important practical problem. The descent direction also matters in this convergence problem. 

While some methods like: Wolfe conditions, through which the finding of local minima is guaranteed.


### # Reference
> https://en.wikipedia.org/wiki/Gradient_descent <br>
> https://en.wikipedia.org/wiki/Wolfe_conditions

### # Keyword Desc

