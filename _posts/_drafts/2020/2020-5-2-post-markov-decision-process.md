---
layout: post
title: "Finite Markov decision process"
subtitle: '马尔可夫决策过程'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-05-02 09:29 
lang: ch 
catalog: true 
categories: documentation
tags:
  - reinforcement learning 
  - Time 2020
---
## Abstract

## 马尔可夫性质
$(S,P)$表示，其中$S$是有限状态集，$P$是状态转移概率矩阵。

## 马尔可夫过程
$(S,A,P,R,\gamma)$，其中$S$为有限状态集合，$A$是有限动作集合，$P$状态转移概率，$R$是回报函数，$\gamma$为折扣因子，用来计算累计回报。

## 马尔可夫决策过程

## vector
强化学习的目标是给定一个马尔科夫决策过程，寻找最优策略。策略是一种从状态到动作的映射关系，常用符号$\pi$表示，可以用给定状态$s$时，动作集上的条件分布表示：

$$\pi(a\mid s)=p[A_t=a\mid S_t=s]$$

因此，强化学习的目标可以看成是，给定系统中任意一个状态，获得当前状态所有可能动作集合中的各个动作的概率，使得最终的累计回报最大。

$$
G_{t}=R_{t+1}+\gamma R_{t+2}+\cdots=\sum_{k=0}^{\infty} \gamma^{k} R_{t+k+1}
$$
## 计算累计回报
强化学习的过程中一般采用随机策略进行，使用随机策略能够在学习过程中，将`探索`过程很好的耦合到`采样`过程，`探索`的主要目的是类似贪心方法找到最优策略，`采样`的过程是进行策略的选择。因此，给定初始状态 ，以及策略 ，状态序列可能有多种情况，因此，

### 累计回报的基本定义
### 累计回报的

<center><img src="/img/in-post/optimizer/saddle.pdf" width="100%"></center>

## Reference


> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内