---
layout: post
title: "Markov Chain Monte Carlo II"
author: "melon"
date: 2022-11-10 21:59
categories: "2022"
tags:
  - math
---
### # Detailed Balance Condition of Markov Chain
Definition: Given aperiodic Markov Chain state transition matrix $$P$$, the probability distribution $$\pi(x)$$, the state set $$S$$, for all $$i,j$$, the following equation holds:

$$ \pi_i P_{ij} = \pi_j P_{ji},\; for\ all\ i,j\in S$$

then the $$\pi$$ is the stationary distribution of transition state matrix $$P$$:

$$\pi P=\pi$$

The condition is a sufficient condition.

<hr>

### # Proofs for Detailed Balanced Condition

sum both side up:

$$ \sum_i\pi_i P_{ij} = \sum_i\pi_j P_{ji} \quad \Rightarrow \quad \sum_i\pi_i P_{ij} = \pi_j \sum_i P_{ji}$$

Obviously, $$ \sum_i P_{ji}=1 $$, thus we have:

$$ \sum_i\pi_i P_{ij} = \pi_i \quad \Rightarrow \quad \pi P=\pi$$

try explain this in details, for each j in state set:

$$ \sum_i\pi_i P_{i1} = \pi_1, \sum_i\pi_i P_{i2}=\pi_2,...,\sum_i\pi_i P_{iN}=\pi_N $$

which is typically another form of $$\pi P=\pi$$.

<hr>

### # Sufficient and Necessary Condition
The detailed balance condition of Markov Chain is a sufficient and necessary condition when it's applyed to 2D distribution.

Proofs:
Given initial state vector (2D): 

$$v=[v_1, v_2]$$

transition state matrix is in the form of: 

$$
P=
\left[\begin{array}{ll}
p_{11} & p_{12} \\
p_{21} & p_{22}
\end{array}\right]
$$

when reaches the stationary condition, we have $$v^t = v^{t+1}$$:

$$ 
v^tP = [v_1^t,v_2^t]
\left[\begin{array}{ll}
p_{11} & p_{12} \\
p_{21} & p_{22}
\end{array}\right]
= [v_1^tp_{11}+v_2^tp_{21}, v_1^tp_{12}+v_2^tp_{22}] = [v_1^{t+1}, v_2^{t+1}]
$$

equivalent to:

$$ v_1^tp_{11}+v_2^tp_{21} = v_1^{t+1}, \quad v_1^tp_{12}+v_2^tp_{22} = v_2^{t+1} $$

when reaching stationary condition, plug $$v_1^{t}=v_1^{t+1}$$ and $$v_2^{t}=v_2^{t+1}$$ in:

$$ v_1^tp_{11}+v_2^tp_{21} = v_1^t, \quad v_1^tp_{12}+v_2^tp_{22} = v_2^t $$

simplify by $$ p_{11}+p_{12}=1$$ and $$p_{21}+p_{22}=1$$, we found that:

$$ v_1^t(1-p_{11}) = v_2^tp_{12} \quad \Rightarrow \quad v_1^tp_{12}=v_2^tp_{12} $$

$$ v_2^t(1-p_{22}) = v_1^tp_{12} \quad \Rightarrow \quad v_2^tp_{21}=v_1^tp_{12} $$

which conform to the detailed balanced condition of Markov Chain.

<hr>

### # Sufficient Condition
The detailed balance condition of Markov Chain is a sufficient condition when it's applyed in a distribution>=2D.

<hr>

### # Markov Chain Monte Carlo Sampling
Thus the MCMC sampling can be narrowed down to the method: try to find a matrix conform to detailed stationary condition, thus we have $$\pi P=\pi$$ which denotes the stationary Markov distribution, finally we could use the derived matrix to generate sampling points.

Unfortunately, hardly can we found matrix $$P$$ which conforms to detailed stationary condition. 

In general, a certain Markov Chain state transition matrix $$Q$$ with target stationary distribution $$\pi$$, does not meet the detailed balance condition:

$$ \pi_iQ_{ij} \ne \pi_jQ_{ji},\ for\ all\ i,j \in S $$

Possibly, we could repair the above formula with $$\alpha$$:

$$ \pi_iQ_{ij}\alpha_{ij} = \pi_jQ_{ji}\alpha_{ji} $$ 

$$ \alpha_{ij}=\pi_jQ_{ji}, \; \alpha_{ji}=\pi_iQ_{ij} $$

Let Markov Chain state transition matrix $$P_{ij} = Q_{ij}\alpha_{ij}$$, thus we could build up the matrix conform to the detailed balance condition. 

Moreover, the $$\alpha_{ij}$$ can be viewed as "acceptance-rejection rate" in the range of $$[0,1]$$. Hence, we could derive Markov Chain state transition matrix $$P$$ by accepting or rejecting a common markov Chain state transition matrix $$Q$$ at certain possibility.

The Markov Chain Monte Carlo sampling method are as follows: <br>
\- input any given Markov Chain state transition matrix $$Q$$, target stationary distribution $$\pi_x$$, maximum transition times $$n$$, number of samples to get $$m$$. <br>
\- derive initial state $$x_0$$ from arbitary simple distribution. <br>
\- for $$t=0$$ to $$n+m-1$$ <br>
&emsp; - sample from condition distribution $$Q(x\mid x_t)$$ to get $$x_*$$ <br>
&emsp; - sample from uniform distribution to get $$u$$ <br>
&emsp; - if $$u<\alpha(x_t,x_*) = \pi(x_*)Q(x_*,x_t)$$, then accept $$x_t \to x_*$$, which denotes $$x_{t+1} = x_*$$ <br>
&emsp; - else reject the transition, $$t=max\{t-1,0\}$$ <br>

<hr>

### # Metropolis-Hastings Sampling
The drawbacks of the above sampling method is that: usually, $$\alpha(x_t,x_*)$$ is extremelly small so that the accept rate of the sampling is fairly small, which may lead to failure in Markov Chain convergence.

We could just ajust the following equation by adding a multiply factor on both side:

$$ \pi_iQ_{ij} \alpha_{ij} = \pi_jQ_{ji} \alpha_{ji} $$ 

the multiplier is as follows to make the above equation holds, and let acceptance rate not that small:

$$ \alpha_{ij}=min\{\frac{\pi_jQ_{ji}}{\pi_iQ_{ij}},1\}, \quad \alpha_{ji}=min\{\frac{\pi_iQ_{ij}}{\pi_jQ_{ji}},1\} $$

The Metropolis-Hastings sampling method are as follows: <br>
\- input any given Markov Chain state transition matrix $$Q$$, target stationary distribution $$\pi_x$$, maximum transition times $$n$$, number of samples to get $$m$$. <br>
\- derive initial state $$x_0$$ from arbitary simple distribution. <br>
\- for $$t=0$$ to $$n+m-1$$ <br>
&emsp; - sample from condition distribution $$Q(x\mid x_t)$$ to get $$x_*$$ <br> 
&emsp; - sample from uniform distribution $$U(0,1)$$to get $$u$$ <br>
&emsp; - if $$u<\alpha(x_t,x_*) = min\{\frac{\pi_*Q(x_*,x_t)}{\pi_tQ(x_t,x_*)}\quad ,1\}$$, then accept $$x_t \to x_*$$, which denotes $$x_{t+1} = x_*$$ <br> 
&emsp; - else reject the transition, $$t=max\{t-1,0\}$$ <br>

In most cases, the chosen Markov Chain transition matrix is symmetrical, the acceptance rate can be written as:

$$\alpha_{ij} = min\{\frac{\pi_j}{\pi_i}, 1\}$$

Moreover, the detailed balance condition can be generalized in vectorized mode:

$$\pi(I)P(I,J) = \pi(J)P(J,I)$$

in which $$I$$ and $$J$$ are vector of multi-dimension.

<hr>

### # M-H sampling python example
Assuming that target distribution as $$N(\mu=3, \sigma=2)$$, the chosen Markov Chain state transition matrix $$Q(i,j)=N(\mu=i,\sigma=1)$$, thus we could using the following python snippet to mimic M-H sampling algorithm.
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
    u = random.uniform(0, 1)                                                  # u from U(0,1)
    x_star = norm.rvs(loc=pi[t - 1], scale=sigma, size=1, random_state=None)  # x_* from Q(x|x_t)
    alpha = min((norm_dist_prob(x_star[0])/norm_dist_prob(pi[t-1])), 1)       # alpha(x_t,x_*)
    pi[t] = x_star[0] if u<alpha else pi[t-1]                                 # iteration

plt.scatter(pi, norm.pdf(pi, loc=3, scale=2), label='')                       # target distribution
plt.hist(pi, num_bins=50, normed=1, facecolor='red', alpha=0.7, label='')     # sample distribution
plt.legend()
plt.show()
```

<hr>

### # Gibbs Sampling
M-H sampling has 2 drawbacks: <br>
<1> Generally, the feature dimension of the sample points are large, which makes the computing of acceptance formula $$min\{\pi_jQ_{ij}/\pi_iQ_{ji}, 1\}$$ time-consuming, moreover, $$\alpha_{ij}<1$$ is a waste of resources. Can we design a method without rejection? <br>
<2> Due to the curse of dimensionality, hardly can we derive the joint distribution of all features, but marginal condition distribution are commonly easy to get. Can we design a sampling method from the known marginal distributions?

Thus, we should find another way to the defailed balance condition.

<hr>

### # 2-D Gibbs Sampling Proofs
Assuming $$\pi(x,y)$$ as 2-d joint distribution, observing two points $$A(x_1,y_1)$$ and $$B(x_1,y_2)$$, the following equation holds:

$$
\pi(x_1, y_1) \pi(y_2 \mid x_1) = \pi(x_1) \pi(y_1 \mid x_1) \pi(y_2 \mid x_1) \\
\pi(x_1, y_2) \pi(y_1 \mid x_1) = \pi(x_1) \pi(y_2 \mid x_1) \pi(y_1 \mid x_1) 
$$

judging by the right side's equality, which gives us:

$$
\pi(x_1, y_1) \pi(y_2 \mid x_1) = \pi(x_1, y_2) \pi(y_1 \mid x_1) \\
\pi(A)\pi(y_2 \mid x_1) = \pi(B)\pi(y_1 \mid x_1) \; \Leftrightarrow \; \pi_iP_{ij}=\pi_jP_{ji}
$$

thus, if the Markov Chain state transition matrix $$P_{i\to j}$$ denotes $$\pi(y\mid x_1)$$, $$i=A,\; j=B$$, the transition between any two points on the line $$x=x_1$$ conforms to the defailed balance condition.

Similarly, given the points $$C(x_2,y_1)$$ with $$A(x_1,y_1)$$, the following equation holds:

$$
\pi(x_1, y_1) \pi(x_2 \mid y_1) = \pi(y_1) \pi(x_1 \mid y_1) \pi(x_2 \mid y_1) \\
\pi(x_2, y_1) \pi(x_1 \mid y_1) = \pi(y_1) \pi(x_2 \mid y_1) \pi(x_1 \mid y_1) 
$$

judging by the right side's equality, which gives us:

$$
\pi(x_1, y_1) \pi(x_2 \mid y_1) = \pi(x_2, y_1) \pi(x_1 \mid y_1) \\
\pi(A)\pi(x_2 \mid y_1) = \pi(C)\pi(x_1 \mid y_1) \; \Leftrightarrow \; \pi_iP_{ij}=\pi_jP_{ji}
$$

thus, if the Markov Chain state transition matrix $$P_{i\to j}$$ denotes $$\pi(x\mid y_1)$$, $$i=A,\; j=C$$, the transition between any two points on the line $$y=y_1$$ conforms to the defailed balance condition.

To sum-up the above observations, we can concluded a way to derive the Markov Chain state transition matrix $$P$$ as:

$$
P(A\to B) = \pi(y_B\mid x_1), \; if\ x_A=x_B=x_1 \\
P(A\to C) = \pi(x_C\mid y_1), \; if\ y_A=y_B=y_1 \\
P(A\to D) = 0, \; in\ other\ conditions
$$

<hr>

### # 2-D Gibbs Sampling Algorithm
\- input stationary distribution $$\pi(x,y)$$, maximum transition times $$n$$, number of samples to get $$m$$. <br>
\- initialize start values $$x_1, y_1$$ randomly. <br>
\- for $$t=0$$ to $$t=n+m-1$$: <br>
&emsp; - sampling from condition distribution $$\pi(y\mid x_t)$$ to get $$y_{t+1}$$. <br>
&emsp; - sampling from condition distribution $$\pi(x\mid y_{t+1})$$ to get $$x_{t+1}$$. <br>
\- during the sampling, the Markov Chain transition are along the 2 axis: $$(x_1,y_1)\to(x_1,y_2)\to(x_2,y_2)...(x_{n+m-1},y_{n+m-1})$$. <br>
\- the derived sample points $$\{(x_{n},y_{n}),(x_{n+1},y_{n+1}),...(x_{n+m-1},y_{n+m-1}) \}$$ is the 2-D Gibbs sampling results. <br>

<hr>

### # 2-D Gibbs Sampling Applied in Guassian Distribution
According to chapter 2.3 of PRML, the 1D form of Guassian distribution are as follows:

$$
\mathcal{N}\left(x \mid \mu, \sigma^2\right)=\frac{1}{\left(2 \pi \sigma^2\right)^{1 / 2}} \exp \left\{-\frac{1}{2 \sigma^2}(x-\mu)^2\right\}
$$

without loss of generality, the multi-variate Gaussian distribution takes the form:

$$
\mathcal{N}(\mathbf{x} \mid \boldsymbol{\mu}, \boldsymbol{\Sigma})=\frac{1}{(2 \pi)^{D / 2}} \frac{1}{|\boldsymbol{\Sigma}|^{1 / 2}} \exp \left\{-\frac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^{\mathrm{T}} \boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})\right\}
$$

in which the $$\boldsymbol{\mu}$$ is mean vector, $$\boldsymbol{\Sigma}$$ is a $$D\times D$$ covariance matrix, and the $$\mid\boldsymbol{\Sigma}\mid$$ denotes its determinant.

Moreover, the Gaussian distribution can be characterized by the `Mahalanobis distance` of $$\mathbf{x}$$ to $$\boldsymbol{\mu}$$:

$$
\Delta^2=(\mathbf{x}-\boldsymbol{\mu})^{\mathrm{T}} \boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})
$$

<details>
    <summary><b>The descriptor for 2D Guassian distribution</b></summary>
    <center><img src="/assets/images/2022/markov-chain-monte-carlo/3.png" width="60%"></center>
</details>

<details>
    <summary><b>The procedure of gibbs sampling applied to Guassian distribution</b></summary>
    <center><img src="/assets/images/2022/markov-chain-monte-carlo/2.gif" width="80%"></center>
</details>

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
    frames = []                                                             # for GIF
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
    plot_gaussian_from_parameters(mean, cov, ax, n_std=2, edgecolor='g', # plot true distribution (Gaussian) by frame
                                  alpha=0.5, label="True Distribution")
    ax.scatter(samples[:num_samples, 0], samples[:num_samples, 1],       # plot already sampled points
               c='b', s=10, label="Sampled Points")
    ax.scatter(samples[0, 0], samples[0, 1], marker='*',
               c='g', s=60, label="Initial Point")
    ax.scatter(tmp_points[:num_tmp, 0], tmp_points[:num_tmp, 1],         # plot newly-added samples from conditional
               c='r', alpha=0.4, s=5, label="Temporary Points")          # distribution
    ax.set_xlim(xlims); ax.set_ylim(ylims)                               # keeping the axes scales same for good GIFS 
    if(num_tmp > 0):                                                     # plot sampling transition lines
        ax.plot([samples[num_samples-1, 0], tmp_points[num_tmp-1, 0]], 
                [samples[num_samples-1, 1], tmp_points[num_tmp-1, 1]], 
                c='k', alpha=0.25)
        if(num_samples > 2):                                             # plot estimated distribution & ignoring
            plot_gaussian_from_points(samples[1:num_samples, 0],         # the starting point -> Gaussian 
                samples[1:num_samples, 1], ax, n_std=2, edgecolor='b', 
                alpha=0.5, label="Estimated Distribution")
    ax.legend(loc='upper left')
    ax.set_title(title)


fig = plt.figure(figsize=(5, 4))                                         # plot the true distribution
ax = fig.gca()
mean = np.array([0, 0]); cov = np.array([[10, 3], [3, 5]])
plot_gaussian_from_parameters(mean, cov, ax, n_std=2, edgecolor='g', 
                              label="True Distribution")
ax.scatter(mean[0], mean[1], c='g')
ax.set_xlim((-11, 11)); ax.set_ylim((-11, 11))
ax.legend(loc='upper left')
# plt.show()

initial_point = [-9.0, -9.0]; num_samples = 500
samples, tmp_points, frames = gibbs_sampler(initial_point, num_samples, 
                                            mean, cov, create_gif=True)

gif.save(frames, "/Users/mac/desktop/gibbs.gif", duration=150)            # GIF
# Image(filename="gibbs.gif")
```

<hr>

### # n-D Gibbs Sampling
\- input stationary distribution $$\pi(x,y,z)$$, set state transition threshold $$n$$, num of samples $$m$$. <br>
\- initialize start values $$(x_1, y_1, z_1)$$. <br>
\- for $$t=0$$ to $$t=n+m+1$$: <br>
&emsp; - sampling from condition distribution $$p(x\mid y_t,z_t)$$ to get $$x_{t+1}$$. <br>
&emsp; - sampling from condition distribution $$p(y\mid x_{t+1},z_{t})$$ to get $$y_{t+1}$$. <br>
&emsp; - sampling from condition distribution $$p(z\mid x_{t+1}, y_{t+1})$$ to get $$z_{t+1}$$. <br>
\- the derived samples $$\{(x_n,y_n,z_n),(x_{n+1},y_{n+1},z_{n+1})...(x_{n+m-1},y_{n+m-1},z_{n+m-1})\}$$ is the n-D Gibbs sampling results. <br>

<hr>

### # Reference
> https://en.wikipedia.org/wiki/Markov_chain_Monte_Carlo <br>
> https://zhuanlan.zhihu.com/p/37121528
> https://mr-easy.github.io/2020-05-21-implementing-gibbs-sampling-in-python/
> https://houbb.github.io/2020/01/28/math-01-markov-chain <br>
> https://www.zhihu.com/question/63305712 <br>
> https://www.cnblogs.com/pinard/p/6632399.html <br>

> https://chaoli.club/index.php/2066/0

<hr>

### # Keyword Desc
`Aperiodic Markov Chain`: The transition state are not in a loop. A periodic Markov Chain is non-convergent.
For any state $$i$$, let $$d$$ denotes the greatest common divisor of set $$\{n\mid n>=1, P_{ii}^n>0\}$$, if $$d=1$$, then this Markov Chain is aperiodic.

`Any two of Markov Chain State are connected`: The possiblility start from arbitary state to any end state within limited steps is non-zero.

`The number of states in Markov Chain`: The number can be infinite of finite, used for continuous probability distribution and discrete probability distribution separately.
