---
layout: post
title: "markov chain monte carlo (I)"
author: "melon"
date: 2022-11-01 22:46
categories: "2022"
tags:
  - math
---

markov chain monte carlo (MCMC) helps generate samples from a given distribution. 
in statistics, MCMC method comprise a class of algorithms for sampling from a probability
distribution. 

by constructing a markov chain that has the desired distribution as its equilibrium distribution,
one can obtain samples of the desired distribution by recording states from the chain. 

MCMC(1) mainly covers the topic: given a transition matrix $P$,
how to do sampling from its relative distribution. 

<hr>

### # markov chain: definition
markov chain assumption: the state of time t only depends on the state of t-1.
let $S_t$ denotes state of time t, the assumption is as follows:

$$ P(S_{t+1}\mid S_1...S_{t-2},S_{t-1},S_{t}) = P(S_{t+1}\mid S_t) $$

<hr>

### # markov chain: a stock example
the stock model has 3 states: bear_market(0), bull_market(1), stagnant-market(2).
let $P(j\mid i)$ represents the probability from state $i$ to state $j$.
the state transition model is as:

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/mcmc.pdf" width="320"/>

the state transition matrix represents the above model is as follows:

$$ 
P=\left(\begin{array}{ccc}
0.9 & 0.075 & 0.025 \\
0.15 & 0.8 & 0.05 \\
0.25 & 0.25 & 0.5
\end{array}\right)
$$

whatever the init state is, the stationary distribution corresponding to certain transition
matrix can be computed as:
```text
import numpy as np

matrix = np.matrix([[0.9, 0.075, 0.025],              # transition matrix
                    [0.15, 0.8, 0.05],
                    [0.25, 0.25, 0.5]], dtype=float)
state = np.matrix([[0.3, 0.4, 0.3]], dtype=float)     # init
for i in range(100):                                  # state transition
    state = state * matrix
    print(f"Current round: {i+1}")
    print(state)
```

<hr>

### # markov chain: convergence conditions
given conditions as follows: for an aperiodic markov chain with state transition matrix $P$,
$P_{ij}$ denotes the transition probability from start state $i$ to end state $j$.
any two states from i to j are connected with each other.
the stationary state $\lim_{n\to\infty}P_{ij}^n$ is independent with start state $i$.
then the following equations are established:

(1) the stationary distribution of markov chain can be described as:

$$ \lim_{n\to \infty} P_{ij}^n=\pi(j) $$

which means the stationary state is independant with start state $i$.

(2) the stable state transition matrix is as follows: 

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

that is, from any start state to certain end state, the transition probability for start->end
is only determined by the end state j.

(3) the probability of state $j$ is computed as:

$$ \pi(j)=\sum_{i=0}^{\infty}\pi(i)P_{ij}$$

(4) $\pi$ is the only non-negative solution of equation $\pi P=\pi$:

$$ \pi = [\pi(1),\pi(2)...\pi(j)...] \quad \sum_{m=0}^{\infty}\pi(m)=1 $$ 

<hr>

### # sampling using markov chain model
if the markov chain state transition matrix P are already known,
then it's easy to obtain the sample of the target stationary distribution.

given the initial probability distribution as $\pi_0(x)$,
after the first markov chain transition, the distribution becomes $\pi_1(x)$.
the i-th distribution will be $\pi_i(x)$, which can be computed as follows:

$$\pi_i(x)=\pi_{i-1}(x)P=\pi_{i-2}P^2=\pi_0(x)P^i$$

assuming from n-th distribution on, the markov chain get stable,
then following equation holds:

$$\pi_n(x)=\pi_{n+1}(x)=...=\pi(x)$$

steps of sampling from markov chain stationary distribution:  
(1) set the markov chain state transition matrix as $P$, set state transition threshold as $n$,
number of sample points as $m$.  
(2) sampling from arbitary common distribution to get initial state $x_0$.  
(3) for t ranges from 0 to n+m-1, sampling from condition distribution
$\pi_t(x)P = \pi_{t+1}(x)$ to get $x_{t+1}$.  
(4) after iteration reaches threshold n, then $x_{n},x_{n+1},...,x_{n+m-1}$ is the samples
from desired stationary distribution.

the more steps (n) included, the more convincible the distribution of the sample matches the
actual desired distribution.

<hr>

### # conclusion
we have discussed about how to use a given state transition matrix $P$ to derive
the sampling points of stationary distribution $\pi(x)$.

however, we haven't solve the problem that: given certain stationary distribution $\pi(x)$,
how can we get its markov chain transition matrix $P$.

MCMC takes a roundabout way to solve this, which will be discussed in
markov chain monte carlo II.

<hr>

### # reference
https://en.wikipedia.org/wiki/Markov_chain_Monte_Carlo  
https://kexue.fm/archives/8084#马尔可夫链  
https://www.cnblogs.com/pinard/p/6632399.html

<hr>

### # keyword desc
aperiodic markov chain: the transition state are not in a loop.
a periodic markov chain is non-convergent.
for any state $i$, let $d$ denotes the greatest common divisor of set
$\{n\mid n>=1, P_{ii}^n>0\}$, if $d=1$, then this markov chain is aperiodic.

any two of markov chain state are connected: the possiblility start from arbitary state to
any end state within limited steps is non-zero.
hence, no isolated island of all possible states.

the number of states in markov chain: the number can be infinite of finite,
used for continuous probability distribution and discrete probability distribution separately.
