---
layout: post
title: "Markov Chain Monte Carlo I"
author: "melon"
date: 2022-11-01 22:46
categories: "2022"
tags:
  - math
---

### # Introduction
MCMC helps generate samples from given distribution. 

In statistics, Markov chain Monte Carlo (MCMC) methods comprise a class of algorithms for sampling from a probability distribution. 

By constructing a Markov chain that has the desired distribution as its equilibrium distribution, one can obtain a sample of the desired distribution by recording states from the chain. 

MCMC(1) mainly covers the topic: given a transition matrix $$P$$, how to do sampling from its relative distribution. 

<hr>

### # Definition of Markov Chain
Markov chain assumes that the state of time t only depends on the state of t-1. Let $$S_t$$ denotes state of time t, the assumption is as follows:

$$ P(S_{t+1}\mid ...S_{t-2},S_{t-1},S_{t}) = P(S_{t+1}\mid S_t) $$

<hr>

### # Markov Chain: a Stock Example
There are 3 state in the stock model: bear_market, bull_market, stagnant-market, denote as state 0-2. Let $$P(j\mid i)$$ represents the probability from state $$i$$ to state $$j$$.

state transition model:
```txt
                    ┌-0.5--┐
                    |      |
              ┌─────┴──────+──────┐
       ┌----->│  stagnant market  │<-----┐
       |      └─┬───────────────┬─┘      |
       |  ┌-----┘               └-----┐  |
  0.025|  |                           |  | 0.05
       |  |0.25                   0.25|  |
       |  |                           |  |
       |  |                           |  |
  ┌────┴──+─────┐                ┌────+──┴─────┐
┌>│ bull market ├-----0.15------>│ bear market │<┐
| └───┬─────+───┘                └──┬──────┬───┘ |
| 0.9 |     └--------0.075----------┘      | 0.8 |
└-----┘                                    └-----┘
```

Then, the state transition matrix represents the above model is as follows:

$$ 
P=\left(\begin{array}{ccc}
0.9 & 0.075 & 0.025 \\
0.15 & 0.8 & 0.05 \\
0.25 & 0.25 & 0.5
\end{array}\right)
$$

Whatever the init state is, the stationary distribution corresponding to a fixed transition matrix can be tested by the python script as follows:
```text
import numpy as np

matrix = np.matrix([[0.9,0.075,0.025],              # transition matrix
                    [0.15,0.8,0.05],
                    [0.25,0.25,0.5]], dtype=float)
state = np.matrix([[0.3,0.4,0.3]], dtype=float)     # init
for i in range(100):                                # state transition
    state = state*matrix
    print(f"Current round: {i+1}")
    print(state)
```

<hr>

### # Markov Chain Convergence Conditions
For an aperiodic Markov Chain with state transition matrix $$P$$, $$P_{ij}$$ denotes the transition probability from start state $$i$$ to end state $$j$$: thus, any two of the states are connected with each other, the stationary state $$\lim_{n\to\infty}P_{ij}^n$$ is independent with start state $$i$$, then the following equations are established:

(I) The stationary distribution of Markov Chain can be described as:

$$ \lim_{n\to \infty} P_{ij}^n=\pi(j) $$

(II) then the stable state transition matrix is as follows: 

$$
\lim_{n\to \infty} P^n = 
\left(
\begin{array}{ccc}
\pi(1) & \pi(2) & ... & \pi(j)\; ...\\
\pi(1) & \pi(2) & ... & \pi(j)\; ...\\
  & & ...& & \\
\pi(1) & \pi(2) & ... & \pi(j)\; ...\\
\end{array}
\right)
$$

(III) the probability of state $$j$$ is computed as:

$$ \pi(j)=\sum_{i=0}^{\infty}\pi(i)P_{ij}$$

(IV) $$\pi$$ is the only none negative solution of the $$\pi P=\pi$$:

$$ \pi = [\pi(1),\pi(2)...\pi(j)...] \quad \sum_{m=0}^{\infty}\pi(m)=1 $$ 

<hr>

### # Sampling based on Markov Chain Model 
If the corresponding Markov Chain state transition matrix to a stationary distribution are found, it's easy to obtain the sample from the distribution.

Given the initial probability distribution as $$\pi_0(x)$$, after the first Markov Chain transition, the distribution becomes $$\pi_1(x)$$, the i-th distribution will be $$\pi_i(x)$$, which can be computed as follows:

$$\pi_i(x)=\pi_{i-1}(x)P=\pi_{i-2}P^2=\pi_0(x)P^i$$

Assume that from n-th distribution on, the Markov Chain get stable, then the following equation holds:

$$\pi_n(x)=\pi_{n+1}(x)=...=\pi(x)$$

Now the sampling from Markov Chain stationary distribution step are as follows: <br>
\- set the Markov Chain state transition matrix $$P$$, set state transition threshold $$n$$, number of sample points $$m$$. <br>
\- sample from arbitary common distribution to get initial state $$x_0$$. <br>
\- let t ranges from 0 to n+m-1; sample from condition distribution $$P(x\mid x_t)$$ to get $$x_{t+1}$$: using property of transition matrix. <br>
\- the derived sample list $$x_{n},x_{n+1},...,x_{n+m-1}$$ is the sampling set of desired stationary distribution. 

The more steps that are included, the more closely the distribution of the sample matches the actual desired distribution.

<hr>

### # Conclusion
We have disscus about how to use a given state transition matrix $$P$$ to derive the sampling points of stationary distribution $$\pi(x)$$. 

But till now we haven't solve the problem that: given arbiary stationary distribution $$\pi(x)$$ to get its Markov Chain transition matrix $$P$$. 
MCMC takes a roundabout way to solve this, which will be introduced in Markov Chain Monte Carlo II. 

<hr>

### # Reference
> https://en.wikipedia.org/wiki/Markov_chain_Monte_Carlo <br>
> https://kexue.fm/archives/8084#马尔可夫链 <br>
> https://www.cnblogs.com/pinard/p/6632399.html

<hr>

### # Keyword Desc
`Aperiodic Markov Chain`: The transition state are not in a loop. A periodic Markov Chain is non-convergent.
For any state $$i$$, let $$d$$ denotes the greatest common divisor of set $$\{n\mid n>=1, P_{ii}^n>0\}$$, if $$d=1$$, then this Markov Chain is aperiodic.

`Any two of Markov Chain State are connected`: The possiblility start from arbitary state to any end state within limited steps is non-zero.

`The number of states in Markov Chain`: The number can be infinite of finite, used for continuous probability distribution and discrete probability distribution separately.
