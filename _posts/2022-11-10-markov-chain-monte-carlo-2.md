---
layout: post
title: "markov chain monte carlo (II)"
author: "melon"
date: 2022-11-10 21:59
categories: "2022"
tags:
  - math
---

this article is a follow-up to the previous MCMC I, which will cover the topic
that how to find the formular of markov chain transition matrix P to converge to the desired
stationary distribution $\pi_(x)$, and finally get samples of the desired distribution.

<hr>

### # detailed balance condition of markov chain
definition: given aperiodic markov chain state transition matrix P,
the probability distribution $\pi(x)$, the state set S, for all i, j,
the following equation holds:

$$ \pi_i P_{ij} = \pi_j P_{ji},\; for\ all\ i,j\in S $$

then the $\pi$ is the stationary distribution of state transition matrix $P$:

$$\pi P=\pi$$

the condition is a sufficient condition.

<hr>

### # why detailed balanced condition is sufficient condition?
sum both side of the detailed balanced equation up:

$$ \sum_i\pi_i P_{ij} = \sum_i\pi_j P_{ji} \quad \Rightarrow \quad \sum_i\pi_i P_{ij} = \pi_j \sum_i P_{ji}$$

obviously, $$ \sum_i P_{ji}=1 $$, thus we have:

$$ \sum_i\pi_i P_{ij} = \pi_i \quad \Rightarrow \quad \pi P=\pi$$

try explain this in details, for each j in state set:

$$ \sum_i\pi_i P_{i1} = \pi_1, \; \sum_i\pi_i P_{i2}=\pi_2\; ... \; \sum_i\pi_i P_{iN}=\pi_N $$

which is typically another form of $\pi P=\pi$.

<hr>

### # sufficient and necessary condition
the detailed balance condition is a sufficient and necessary condition
only when it's applyed to 2d distribution.
proof of the necessity of detailed balanced conditon to a stationary markov distribution is
concluded as following equations.

given initial state vector (2d) as:

$$v=[v_1, v_2]$$

transition state matrix is in the form of: 

$$
P=
\left[\begin{array}{ll}
p_{11} & p_{12} \\
p_{21} & p_{22}
\end{array}\right]
$$

when reaching the stationary condition, we have $v^t = v^{t+1}$:

$$ 
v^tP = [v_1^t,v_2^t]
\left[\begin{array}{ll}
p_{11} & p_{12} \\
p_{21} & p_{22}
\end{array}\right]
= [v_1^tp_{11}+v_2^tp_{21}, \; v_1^tp_{12}+v_2^tp_{22}] = [v_1^{t+1}, v_2^{t+1}]
$$

equivalent to:

$$ v_1^tp_{11}+v_2^tp_{21} = v_1^{t+1}, \quad v_1^tp_{12}+v_2^tp_{22} = v_2^{t+1} $$

when reaching stationary condition, plug $v_1^{t}=v_1^{t+1}$ and $v_2^{t}=v_2^{t+1}$ in:

$$ v_1^tp_{11}+v_2^tp_{21} = v_1^t, \quad v_1^tp_{12}+v_2^tp_{22} = v_2^t $$

simplify the above formular using $p_{11}+p_{12}=1$ and $p_{21}+p_{22}=1$, found that:

$$ v_1^t(1-p_{11}) = v_2^tp_{12} \quad \Rightarrow \quad v_1^tp_{12}=v_2^tp_{12} $$

$$ v_2^t(1-p_{22}) = v_1^tp_{12} \quad \Rightarrow \quad v_2^tp_{21}=v_1^tp_{12} $$

which conform to the detailed balanced condition of markov chain.

<hr>

### # sufficient condition
the detailed balance condition of markov chain is a sufficient condition when it's applyed in
a distribution>=2d.

<hr>

### # markov chain monte carlo sampling
given detailed balanced condition above, MCMC sampling can be narrowed down to:  
try find a matrix conform to detailed stationary condition, by which $\pi P=\pi$ denotes
the stationary markov distribution. then the derived matrix can be used to generate
sampling points.

unfortunately, hardly can we found matrix $P$ which conforms to detailed stationary condition.
we can resort to the idea of acceptance-rejection sampling, to derive the final result by
optimizing a easy-to-get transition matrix.

in general cases, a certain markov chain state transition matrix $Q$ with
target stationary distribution $\pi$, does not meet the detailed balance condition:

$$ \pi_iQ_{ij} \ne \pi_jQ_{ji},\ for\ all\ i,j \in S $$

possibly, we can repair the above formula with $\alpha$:

$$ \pi_iQ_{ij}\alpha_{ij} = \pi_jQ_{ji}\alpha_{ji} $$ 

it's quite easy to conclude that the following scheme meet the need:

$$ \alpha_{ij}=\pi_jQ_{ji}, \; \alpha_{ji}=\pi_iQ_{ij} $$

thus, let markov chain state transition matrix $P_{ij} = Q_{ij}\alpha_{ij}$,
thus we can build up the matrix conform to the detailed balance condition.  
moreover, the $\alpha_{ij}$ can be seen as "acceptance-rejection rate" in the range of
$[0,1]$.
hence, we could derive markov chain state transition matrix $P$ by accepting or rejecting
a common markov chain state transition matrix $Q$ at certain possibility.

the markov chain monte carlo sampling method are as follows:  
(1) input any given markov chain state transition matrix $Q$, target stationary distribution
$\pi(x)$, maximum transition times $n$, number of samples to get $m$.  
(2) derive initial state $x_0$ from arbitary simple distribution.  
(3) for $t=0$ to $n+m-1$:  
(3.1) sampling from condition distribution $Q(x\mid x_t)$ to get $x_\*$  
(3.2) sampling from uniform distribution to get $u$  
(3.3) if $u<\alpha(x_t,x_\*) = \pi(x_\*)Q(x_\*,x_t)$, then accept $x_t \to x_\*$,
which denotes $x_{t+1} = x_\*$  
(3.4) else reject the transition, $x_{t+1}=x_t$ ($t=max\left(t-1,0\right)$).  

<hr>

### # metropolis-hastings sampling
the drawbacks of the above sampling method is that:
usually, $\alpha(x_t,x_\*)$ is extremely small, thus the accept rate of the sampling
is fairly small, which makes it hard for markov chain to convergence.

recall the MCMC sampling equation:

$$ \pi_iQ_{ij} \alpha_{ij} = \pi_jQ_{ji} \alpha_{ji} $$

by adding a multiply factor on both side, the M-H sampling alleviate the MCMC drawbacks:

$$ \alpha_{ij}=min\{\frac{\pi_jQ_{ji}}{\pi_iQ_{ij}},1\}, \quad \alpha_{ji}=min\{\frac{\pi_iQ_{ij}}{\pi_jQ_{ji}},1\} $$

the multiplier above is designed to make the detailed balanced condition holds,
and to make sure the acceptance rate considerable.

the metropolis-hastings sampling method are as follows:  
(1) input any given markov chain state transition matrix $Q$, target stationary distribution
$\pi_x$, maximum transition times $n$, number of samples to get $m$.  
(2) derive initial state $x_0$ from arbitary simple distribution.  
(3) for $t=0$ to $n+m-1$:  
(3.1) sampling from condition distribution $Q(x\mid x_t)$ to get $x_\*$  
(3.2) sampling from uniform distribution $U(0,1)$ to get $u$  
(3.3) if $u<\alpha(x_t,x_\*) = min\left(\frac{\pi_\*Q(x_\*,x_t)}{\pi_tQ(x_t,x_\*)}\quad ,1\right)$,
then accept $x_t \to x_\*$, which denotes $x_{t+1} = x_\*$  
(3.4) else reject the transition, $x_{t+1}=x_t$ ($t=max\left(t-1,0\right)$).

in most cases, the chosen markov chain transition matrix is symmetrical:

$$*Q(x_*,x_t) = Q(x_t,x_*)$$

thus, according to (3.3), the acceptance rate can be re-written as:

$$\alpha_{ij} = min\{\frac{\pi_j}{\pi_i}, 1\}$$

moreover, the detailed balance condition can be generalized in vectorized mode:

$$\pi(I)P(I,J) = \pi(J)P(J,I)$$

in which $I$ and $J$ are multi-dimension vectors.

<hr>

### # metropolis-hastings sampling code (python) 
assuming target distribution as $N(\mu=3, \sigma=2)$,
the chosen markov chain state transition matrix $Q(i,j)=N(\mu=i,\sigma=1)$,
thus the following python snippet can be used to mimic M-H sampling algorithm.

```text
import matplotlib.pyplot as plt
from scipy.stats import norm

def norm_dist_prob(theta):                                                    # target stationary distribution -> \pi
    y = norm.pdf(theta, loc=3, scale=2)
    return y

T = 5000
pi = [0 for i in range(T)]
sigma = 1
t = 0
while t < T-1:
    t = t + 1
    u = random.uniform(0, 1)                                                  # u ~ U(0,1)
    x_star = norm.rvs(loc=pi[t - 1], scale=sigma, size=1, random_state=None)  # x_* ~ Q(x|x_t)
    alpha = min((norm_dist_prob(x_star[0])/norm_dist_prob(pi[t-1])), 1)       # alpha(x_t,x_*)
    pi[t] = x_star[0] if u<alpha else pi[t-1]                                 # accept or reject

plt.scatter(pi, norm.pdf(pi, loc=3, scale=2), label='')                       # target distribution
plt.hist(pi, num_bins=50, normed=1, facecolor='red', alpha=0.7, label='')     # sample distribution
plt.legend()
plt.show()
```

<hr>

### # drawbacks of M-H sampling
(1) generally, the feature dimension of the sample points are large,
which makes the computing of acceptance formula $min\left(\pi_jQ_{ij}/\pi_iQ_{ji}, 1\right)$
time-consuming, moreover, $\alpha_{ij}<1$ is a waste of resources.
can we design a method without rejection?

(2) due to the curse of dimensionality, hardly can we derive the joint distribution of
all features, but marginal condition distribution are commonly easier to get.
can we design a sampling method from the known marginal distributions?

that is to say, we should find another way to conform to the defailed balance condition,
and optimize the MC sampling steps.

<hr>

### # gibbs sampling
analysis can be started on simple 2d joint distribution $\pi(x,y)$.
observations can be made on two points $A(x_1,y_1)$ and $B(x_1,y_2)$ (the first dimension are the same),
then the following equation holds:

$$
\pi(A)\; \pi(y_2 \mid x_1) = \pi(x_1, y_1)\; \pi(y_2 \mid x_1) = \pi(x_1) \pi(y_1 \mid x_1)\; \pi(y_2 \mid x_1) \\
\pi(B)\; \pi(y_2 \mid x_1) = \pi(x_1, y_2)\; \pi(y_1 \mid x_1) = \pi(x_1) \pi(y_2 \mid x_1)\; \pi(y_1 \mid x_1) 
$$

re-organize the above rightmost part of the above equation:

$$ \pi(x_1, y_1)\; \pi(y_2 \mid x_1) = \pi(x_1, y_2)\; \pi(y_1 \mid x_1) $$

there is no harm in setting markov chain state transition matrix
($P_{i\to j}$) as $\pi(y\mid x_1)$, the start & end state as $i=A,\; j=B$, which gives:

$$ \pi(A)\; \pi(y_2 \mid x_1) = \pi(B)\; \pi(y_1 \mid x_1) \; \Leftrightarrow \; \pi_iP_{ij}=\pi_jP_{ji} $$

the transition between any 2 points on the line $x=x_1$
conforms to the defailed balance condition.  
similarily, given the points $C(x_2,y_1)$ with $A(x_1,y_1)$, the following equation holds:

$$
\pi(x_1, y_1) \pi(x_2 \mid y_1) = \pi(y_1) \pi(x_1 \mid y_1) \pi(x_2 \mid y_1) \\
\pi(x_2, y_1) \pi(x_1 \mid y_1) = \pi(y_1) \pi(x_2 \mid y_1) \pi(x_1 \mid y_1) 
$$

re-organize the above rightmost part of the above equation:

$$ \pi(x_1, y_1) \pi(x_2 \mid y_1) = \pi(x_2, y_1) \pi(x_1 \mid y_1) $$

there is no harm in setting markov chain state transition matrix
($P_{i\to j}$) as $\pi(x\mid y_1)$, the start & end state as $i=A,\; j=C$, which gives:

$$ \pi(A)\pi(x_2 \mid y_1) = \pi(C)\pi(x_1 \mid y_1) \; \Leftrightarrow \; \pi_iP_{ij}=\pi_jP_{ji} $$

the transition between any 2 points on the line $y=y_1$ conforms to detailed balance condition.

finally, a shortcut transition path is concluded to always fullfil the detailed balanced condition,
thus providing a algorithm to derive the markov chain state transition matrix P:

$$
P(A\to B) = \pi(y_B\mid x_1), \; if\ x_A=x_B=x_1 \\
P(A\to C) = \pi(x_C\mid y_1), \; if\ y_A=y_B=y_1 \\
P(A\to D) = 0, \; in\ other\ conditions
$$

<hr>

### # 2d gibbs sampling algorithm
(1) input stationary distribution $\pi(x,y)$, maximum transition times $n$,
number of samples to get $m$.  
(2) initialize startup values $x_1, y_1$ randomly.  
(3) for $t=0$ to $t=n+m-1$:  
(3.1) sampling from condition distribution $\pi(y\mid x_t)$ to get $y_{t+1}$.  
(3.2) sampling from condition distribution $\pi(x\mid y_{t+1})$ to get $x_{t+1}$.  
(4) during the sampling, the markov chain transition are along the 2 axis:

$$(x_1,y_1)\to(x_1,y_2)\to(x_2,y_2)...(x_{n+m-1},y_{n+m-1})$$

(5) the derived sample points $\{(x_{n},y_{n}),(x_{n+1},y_{n+1}),...(x_{n+m-1},y_{n+m-1})\}$
are results of gibbs sampling.

<hr>

### # 2d gibbs sampling applied in guassian distribution
according to chapter 2.3 of PRML, the 1d form of gaussian distribution are as follows:

$$
\mathcal{N}\left(x \mid \mu, \sigma^2\right)=\frac{1}{\left(2 \pi \sigma^2\right)^{1 / 2}} \exp \left\{-\frac{1}{2 \sigma^2}(x-\mu)^2\right\}
$$

without loss of generality, the multi-variate gaussian distribution takes the form:

$$
\mathcal{N}(\mathbf{x} \mid \boldsymbol{\mu}, \boldsymbol{\Sigma})=\frac{1}{(2 \pi)^{D / 2}} \frac{1}{|\boldsymbol{\Sigma}|^{1 / 2}} \exp \left\{-\frac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^{\mathrm{T}} \boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})\right\}
$$

in which the $\boldsymbol{\mu}$ is mean vector, $\boldsymbol{\Sigma}$ is a $D\times D$
covariance matrix, and the $\mid\boldsymbol{\Sigma}\mid$ denotes its determinant.

moreover, the gaussian distribution can be characterized by the mahalanobis distance of
$\mathbf{x}$ to $\boldsymbol{\mu}$:

$$
\Delta^2=(\mathbf{x}-\boldsymbol{\mu})^{\mathrm{T}} \boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})
$$

the descriptor of 2d gaussian distribution is as:

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/mcmc3.pdf" width="320"/>

<details>
    <summary><b>the procedure of gibbs sampling applied to gaussian distribution</b></summary>
    <center><img src="/assets/images/2022/markov-chain-monte-carlo/2.gif" width="80%"></center>
</details>

python code for generating the above gif:
```text
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.transforms as transforms
import gif

from matplotlib.patches import Ellipse
from IPython.display import Image
from random import random

def plot_gaussian_from_points(x, y, ax, n_std=3.0, facecolor='none', **kwargs):
    """
    Create a plot of the covariance confidence ellipse of *x* and *y*.
    x, y: array-like, shape (n, )
    ax: matplotlib.axes.Axes. The axes object to draw the ellipse.
    n_std: number of standard deviations to determine the ellipse's radiuses.
    return: matplotlib.patches.Ellipse
    kwargs : matplotlib.patches.Patch properties
    """
    if len(x) != len(y):
        raise ValueError("x and y must be the same size")
    if len(x) < 2:
        raise ValueError("Need more data.")
    cov = np.cov(x, y)                                                       # get eigenval of 2D dataset as
    pearson = cov[0, 1]/np.sqrt(cov[0, 0] * cov[1, 1])                       # radius of ellipse
    ell_radius_x = np.sqrt(1 + pearson)
    ell_radius_y = np.sqrt(1 - pearson)
    ellipse = Ellipse((0, 0), width=ell_radius_x*2, height=ell_radius_y*2,   # set ellipse properties 
        facecolor=facecolor, **kwargs)
    scale_x = np.sqrt(cov[0, 0])*n_std; mean_x = np.mean(x)                  # calculating the std deviation of x/y 
    scale_y = np.sqrt(cov[1, 1])*n_std; mean_y = np.mean(y)
    transf = transforms.Affine2D().rotate_deg(45).scale(scale_x, scale_y) \  # operate transform to ellipse  
        .translate(mean_x, mean_y)
    ellipse.set_transform(transf + ax.transData)
    return ax.add_patch(ellipse)


def plot_gaussian_from_parameters(mean, cov, ax, n_std=3.0, 
                                  facecolor='none', **kwargs):
    """
    Create a plot of the covariance confidence ellipse of *x* and *y*.
    mean: array-like, shape (2, ), Mean vector
    cov: array-like, shape (2,2), Covariance matrix
    ax: matplotlib.axes.Axes, axes object to draw the ellipse.
    n_std: the num of standard deviations to determine the ellipse's radiuses.
    return: matplotlib.patches.Ellipse
    kwargs: matplotlib.patches.Patch properties
    """
    if len(mean) != 2:
        raise ValueError("Mean vector length should be 2.")
    if (cov.shape != (2, 2)):
    	raise ValueError("Covariance should be a 2x2 matrix.")
    if(cov[0, 1] != cov[1, 0]):
        raise ValueError("Covariance should be symmetric.")
    if(cov[0, 0] < 0 or cov[0, 0]*cov[1,1] - cov[0,1]**2 < 0):
        raise ValueError("Covariance should be positive semidefinite.")
    pearson = cov[0, 1]/np.sqrt(cov[0, 0] * cov[1, 1])                       # get eigenval of 2D dataset as 
    ell_radius_x = np.sqrt(1 + pearson)                                      # radius of the ellipse
    ell_radius_y = np.sqrt(1 - pearson)
    ellipse = Ellipse((0, 0), width=ell_radius_x*2, height=ell_radius_y*2,   # set ellipse properties 
        facecolor=facecolor, **kwargs)
    scale_x = np.sqrt(cov[0, 0])*n_std; mean_x = mean[0]                     # calculating the std deviation of x/y 
    scale_y = np.sqrt(cov[1, 1])*n_std; mean_y = mean[1]
    transf = transforms.Affine2D().rotate_deg(45).scale(scale_x, scale_y) \  # operate transform to ellipse  
        .translate(mean_x, mean_y)
    ellipse.set_transform(transf + ax.transData)
    return ax.add_patch(ellipse)


def conditional_sampler(sampling_index, current_x, mean, cov):
    conditioned_index = 1 - sampling_index
    a = cov[sampling_index, sampling_index]
    b = cov[sampling_index, conditioned_index]
    c = cov[conditioned_index, conditioned_index]
    mu = mean[sampling_index] \
        + (b * (current_x[conditioned_index] - mean[conditioned_index]))/c
    sigma = np.sqrt(a-(b**2)/c)
    new_x = np.copy(current_x)
    new_x[sampling_index] = np.random.randn()*sigma + mu
    return new_x


def gibbs_sampler(initial_point, num_samples, mean, cov, create_gif=True):
    frames = []                                                             # for gif
    point = np.array(initial_point)
    samples = np.empty([num_samples+1, 2])                                  # sampled points
    samples[0] = point
    tmp_points = np.empty([num_samples, 2])                                 # inbetween points
    for i in range(num_samples):                                            # iteration
        print(i)
        point = conditional_sampler(0, point, mean, cov)                    # sample from p(x|y)
        tmp_points[i] = point
        if(create_gif):
            frames.append(plot_samples(samples, i+1, tmp_points, i+1, 
            title="Num Samples: " + str(i)))
        point = conditional_sampler(1, point, mean, cov)                    # sample from p(y|x)
        samples[i+1] = point
        if(create_gif):
            frames.append(plot_samples(samples, i+2, tmp_points, i+1, 
            title="Num Samples: " + str(i+1)))
    if(create_gif):
        return samples, tmp_points, frames
    else:
        return samples, tmp_points


@gif.frame
def plot_samples(samples, num_samples, tmp_points, num_tmp, 
                 title="Gibbs Sampling", xlims=(-11, 11), ylims=(-11, 11)):
    fig = plt.figure(figsize=(20, 16))
    ax = fig.gca()
    plot_gaussian_from_parameters(mean, cov, ax, n_std=2, edgecolor='g',  # plot true distribution (Gaussian) by frame
                                  alpha=0.5, label="True Distribution")
    ax.scatter(samples[:num_samples, 0], samples[:num_samples, 1],        # plot already sampled points
               c='b', s=10, label="Sampled Points")
    ax.scatter(samples[0, 0], samples[0, 1], marker='*',
               c='g', s=60, label="Initial Point")
    ax.scatter(tmp_points[:num_tmp, 0], tmp_points[:num_tmp, 1],          # plot newly-added samples from conditional
               c='r', alpha=0.4, s=5, label="Temporary Points")           # distribution
    ax.set_xlim(xlims); ax.set_ylim(ylims)                                # keeping the axes scales same for good GIFS 
    if(num_tmp > 0):                                                      # plot sampling transition lines
        ax.plot([samples[num_samples-1, 0], tmp_points[num_tmp-1, 0]], 
                [samples[num_samples-1, 1], tmp_points[num_tmp-1, 1]], 
                c='k', alpha=0.25)
        if(num_samples > 2):                                              # plot estimated distribution & ignoring
            plot_gaussian_from_points(samples[1:num_samples, 0],          # the starting point -> Gaussian 
                samples[1:num_samples, 1], ax, n_std=2, edgecolor='b', 
                alpha=0.5, label="Estimated Distribution")
    ax.legend(loc='upper left')
    ax.set_title(title)


fig = plt.figure(figsize=(5, 4))                                          # plot the true distribution
ax = fig.gca()
mean = np.array([0, 0]); cov = np.array([[10, 3], [3, 5]])
plot_gaussian_from_parameters(mean, cov, ax, n_std=2, edgecolor='g', label="True Distribution")
ax.scatter(mean[0], mean[1], c='g')
ax.set_xlim((-11, 11)); ax.set_ylim((-11, 11))
ax.legend(loc='upper left')
# plt.show()

initial_point = [-9.0, -9.0]; num_samples = 500
samples, tmp_points, frames = gibbs_sampler(initial_point, num_samples, mean, cov, create_gif=True)

gif.save(frames, "/Users/mac/desktop/gibbs.gif", duration=150)            # gif
```

<hr>

### # n-dimension gibbs sampling
@ 1 input stationary distribution $\pi(x,y,z)$, set state transition threshold $n$,
num of samples $m$.  
@ 2 initialize start values $(x_1, y_1, z_1)$.  
@ 3 for $t=0$ to $t=n+m+1$:  
3.1 sampling from condition distribution $p(x\mid y_t,z_t)$ to get $x_{t+1}$.  
3.2 sampling from condition distribution $p(y\mid x_{t+1},z_{t})$ to get $y_{t+1}$.  
3.3 sampling from condition distribution $p(z\mid x_{t+1}, y_{t+1})$ to get $z_{t+1}$.  
@ 4 the derived samples $\{(x_n,y_n,z_n),(x_{n+1},y_{n+1},z_{n+1})...(x_{n+m-1},y_{n+m-1},z_{n+m-1})\}$
is the n-d gibbs sampling results.

<hr>

### # reference
https://en.wikipedia.org/wiki/Markov_chain_Monte_Carlo  
https://zhuanlan.zhihu.com/p/37121528  
https://mr-easy.github.io/2020-05-21-implementing-gibbs-sampling-in-python/
https://houbb.github.io/2020/01/28/math-01-markov-chain  
https://www.zhihu.com/question/63305712  
https://www.cnblogs.com/pinard/p/6632399.html

<hr>

### # keyword desc
aperiodic markov chain: the transition state are not in a loop. aperiodic markov chain is non-convergent.
for any state $i$, let $d$ denotes the greatest common divisor of set $\{n\mid n>=1, P_{ii}^n>0\}$,
if $d=1$, then this markov chain is aperiodic.

any two of markov chain state are connected:
the possiblility start from arbitary state to any end state within limited steps is non-zero.

the number of states in markov chain can be infinite of finite,
for continuous probability distribution and discrete probability distribution separately.
