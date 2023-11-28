---
layout: post
title: "Word2vec"
subtitle: 'Word to vec embeddings'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-03-18 22:05 
lang: ch 
catalog: true 
categories: documentation
tags:
  - NLP
  - Time 2020
---
## Introduction 
本文主要针对nlp问题中词向量嵌入方式进行简单介绍，将相关知识进行整理，便于知识回顾。本文主要基于cs224d教程进行翻译整理。

## Background
nlp自然语言处理主要通过设计算法的方式让计算机理解人类的自然语言，通常可以分为如下几种应用场景：<br>
`easy` 拼写检查、关键词索引、查找同义词 <br>
`medium` 解析网站或文档信息 <br>
`hard` 机器翻译(Translate Chinese text to English)、语义分析(What is the meaning of query statement?)、指代歧义(What does "he" or "it" refer to given a document?)、机器问答 <br>

无论什么应用场景，都需要将赐予表示成模型可接受的输入，即如何计算词汇之间的相似度。词向量恰能解决上述问题。

英语中有1300万个单词，大部分单词之间存在联系，如猫科动物⟺猫，酒店⟺汽车旅馆等，我们想要将单词编码成为一个向量，使得所有的词汇可以用`单词空间`中的点来表示。最简单的方案是，构建一个1300万维度的词空间，这样，所有词汇将很容易进行编码区分，然而这种编码方案将带来巨大的计算成本，而且不能够很好的描述单词之间的关系，因此，更好的方案是，构建一个$N\ll 1300万$维度的空间，对于所有单词的`语义空间`进行充分表述：例如构建一个3维空间$(time,level,gender)$，则$king(now,highest,male)$、$queen(now,highest,female)$、$prince(future,highest,male)$三个样本点可以在构建的空间中的点予以表示。

如果采用one-hot编码方式，则可以将所有单词进行编码，如下图所示：

<center><img src="/img/in-post/word2vec/onehot.pdf" width="40%"></center>

但是这种单词编码方式将导致无法正确计算单词之间的相似度，因此我们需要降低词向量空间维度，将所有的单词映射到一个可以计算相似度的自空间上：

$$(w^{hotel})^T w^{motel}=(w^{feline})^T w^{cat}=0$$

## SVD Based Methods
> we first loop over a massive dataset and accumulate word `co-occurrence` counts in some form of a matrix $X$, and then perform `Singular Value Decomposition` on $X$ to get a $USV^T$ decomposition. We then use the rows of $U$ as the word embeddings for all words in our dictionary. Let us discuss a few choices of $X$.

下面介绍几种产生$X$矩阵的方法：

**➤ Word-Document Matrix** <br>
基于单词-文档共现矩阵：首先做出一个大胆的假设，即相关的词通常会出现在同一文档中。例如，“银行”，“债券”，“股票”，“货币”等可能会同时出现。 但是“银行”，“章鱼”，“香蕉”和“曲棍球”可能不会始终出现在一起。 我们以下方式构建单词文档矩阵$X$：循环遍历数十亿个文档，每当单词$i$在文档$j$中出现时，我们就在条目$X_{ij}$中添加一个。 显然，这是一个非常大的矩阵$(R^{|Vocab|×M})$，并随文档数$M$缩放。 因此，我们需要更好的方式构建$X$矩阵。

**➤ Window based Co-occurrence Matrix** <br>
基于窗内频次共现矩阵：我们计算每个单词出现在感兴趣单词周围特定大小的窗口内的次数。通过这种方式，计算语料库中所有单词的计数。 假设我们的语料库仅包含三个句子，窗口大小为$1$：<br>
`1` **I enjoy flying.** <br>
`2` **I like NLP.** <br>
`3` **I like deep learning.** <br>
<center><img src="/img/in-post/word2vec/coocurrence_matrix.pdf" width="80%"></center>

在获得上述矩阵$X$之后，进行SVD降维。具体地：对于$X$实施奇异值分解，观察对角矩阵$S$中获得的奇异值，并根据下面的公式确定$k$值，即在何处截断向量：

$$
\text{amount of variance captured by first k dimensions} = \frac{\sum_{i=1}^k \sigma_i}{\sum_{i=1}^{|V|} \sigma_i}
$$

<center><img src="/img/in-post/word2vec/svd.pdf" width="60%"></center>

`Word-Document Matrix`和`Window based Co-occurrence Matrix`方法提供了两种语义、词性进行编码的方式，但是仍然存在一些问题：

`1` 矩阵的尺寸经常变化 - 新单词的添加非常频繁，语料库大小经常变化，因此不得不重新进行词向量构建。<br>
`2` 大多数单词不会同时出现，所得到的矩阵非常稀疏，这使得实际运算效率很低<br>
`3` 矩阵通常具有很高的维度$(10^6,10^6)$，计算复杂度很高 <br>
`4` 需要分两步得到结果 (form x and then perform SVD) <br>
`5` 需要对于$X$矩阵进行额外操作，以解决单词频率出现不均衡对于构建$X$矩阵的。 

针对上述问题，部分解决方案如下：<br>
`1` 忽略诸如"the"、"he"、"has"等功能词。 <br>
`2` 应用ramp window来根据文档中单词和感兴趣单词之间的距离对于co-occurrence计数进行加权计算。 <br>
`3` 使用pearson correlation coefficient对于词汇之间相关性进行描述(代替共现频次计数)，并将相关性系数为负数的置0。 <br>
> As we see in the next section, iteration based methods solve many of these issues in a far more elegant manner.

## Iteration Based Methods
不同于上述方法中一次性计算、存储整个语料库中所有的信息(可能需要计算数十亿量级的语句)，我们可以尝试建立一种能够自动学习(参数更新)词汇编码模型，通过逐步学习的方式解决问题。

### Unigram & Bigrams
这部分内容主要介绍Language Models (Unigrams, Bigrams, etc.)。我们希望所构建的词汇编码模型接受向量化的词汇，并输出其合理的出现概率。以下面的句子为例，**"The cat jumped over the puddle."**：一个训练良好的语言模型将对于这句话输出较高的概率值，因为这句话从句法、语法上来说是合理有效的。因此，**"stock boil fish is toy."**应当输出较低的概率，因为这句话不合乎规则。

一种方案是，给出**Unigram model**如下形式：

$$
P(w_1,w_2,\cdots ,w_n)=\prod_{i=1}^{n}p(w_i)
$$

上述的模型构建方式有点ludicrous，这种方式忽略了单词之间的关系，即词组之间高度的`共现性`，从而导致上述模型可能会对 **"stock boil fish is toy."** 打出较高的分数。另外一种模型能够考虑单词及其相邻单词之间的pairwise probability，称之为**Bigram model**：

$$
P(w_1,w_2,\cdots ,w_n)=\prod_{i=2}^{n}p(w_i\mid w_{i-1})
$$

这种方案也有一点naive，因为这种方案过度的考量了相邻词汇之间的`co-ocurrence`概率，忽略了单词在整句话中的语义、句法的合理性。

### CBOW
> CBOW(Continuous Bag of Words Model) Model - Predicting a center word form the surrounding context

对于输入："The cat _ over the puddle."，根据中心词周围的context语境，通过训练一个模型输出其ground truth中心词jump。经过整个语料库进行充分模型训练，整个模型构建的隐藏层所表示的向量即为词向量，获得方式是输入的词汇乘权重矩阵得到的隐藏层向量为对应的word embedding。

<center><img src="/img/in-post/word2vec/cbow.pdf" width="80%"></center>

cbow模型优化的目标函数是交叉熵，使模型输出的向量分布尽可能接近真实分布，因此，最小化两个分布之间的交叉熵$H(\hat{y},y)=-y_ilog(\hat{y_i})$，$(\hat{y_i})$是模型输出值，又有$label=y_i=1$得到如下公式：

$$
\text {minimize} J =-\log P\left(w_{c} \mid w_{c-m}, \ldots, w_{c-1}, w_{c+1}, \ldots, w_{c+m}\right) =-\log P\left(u_{c} \mid \hat{v}\right) \\ 
=-\log \frac{\exp \left(u_{c}^{T} \hat{v}\right)}{\sum_{j=1}^{\mid V\mid} \exp \left(u_{j}^{T} \hat{v}\right)} = -u_{c}^{T} \hat{v}+\log \sum_{j=1}^{\mid V\mid} \exp \left(u_{j}^{T} \hat{v}\right)
$$

### Skip-Gram Model <br>
> Predicting surrounding context words given a center word.

skip-gram模型恰好和cbow模型的思路相反，给定中心词jump作为输入，模型输出中心词的context单词the cat over the puddle。

<center><img src="/img/in-post/word2vec/skip_gram.pdf" width="80%"></center>

skip-gram模型损失函数正好和cbow模型的损失函数相反，如下所示：

$$
\text { minimize } J =-\log P\left(w_{c-m}, \ldots, w_{c-1}, w_{c+1}, \ldots, w_{c+m} \mid w_{c}\right) \\ 

=-\log \prod_{j=0, j \neq m}^{2 m} P\left(w_{c-m+j} \mid w_{c}\right) =-\log \prod_{j=0, j \neq m}^{2 m} P\left(u_{c-m+j} \mid v_{c}\right) \\ 

=-\log \prod_{j=0, j \neq m}^{2 m} \frac{\exp \left(u_{c-m+j}^{T} v_{c}\right)}{\sum_{k=1}^{\mid V\mid} \exp \left(u_{k}^{T} v_{c}\right)} =-\sum_{j=0, j \neq m}^{2 m} u_{c-m+j}^{T} v_{c}+2 m \log \sum_{k=1}^{\mid V\mid} \exp \left(u_{k}^{T} v_{c}\right)
$$

## Optimization methods
### Negative Sampling
> For every training step, instead of looping over the entire vocabulary, we can just sample several negative examples! 

根据上述cbow模型以及skip-gram模型的思路，计算损失时，softmax分母会将语料库中所有的背景词作为分母，因此，每次的梯度计算开销过大，本节介绍的方法能够很好的近似解决这一问题。这种方法的思想和目标识别中`hard negative mining`有相似之处。

<center><img src="/img/in-post/word2vec/negative_sampling0.pdf" width="80%"></center>

具体负采样的方法如下图所示，在根据单词在语料库中的频次计算$I_i$的长度时，使用右边$count^{3/4}$的形式，根据CS 224D note1中的描述，这种形式有如下的优势：

$$
is: 0.9^{3/4} = 0.92 \quad Constitution: 0.09^{3/4} = 0.16 \quad bombastic: 0.01^{3/4} = 0.032
$$

即这种$频次^{3/4}$的形式，能够相对扩大频次较少的单词出现的比重，提升其被采样的概率，当采样频次远低于语料库容量时使该单词能够发挥一定的作用。具体采样法如下图：

<center><img src="/img/in-post/word2vec/negative_sampling1.pdf" width="80%"></center>

下面对于采样negative sampling方法的损失函数进行推导：<br>
给定中心词$w_c$的一个背景窗口，窗口大小为$m$，我们把背景词$w_o$出现在该背景窗口看作一个事件，并将该事件的概率计算为$P(D=1\mid w_c, w_o)$，同样，可以将背景词$w_o$在corpus中没有出现在中心词$w_c$周围的窗口的概率记为$P(D=0\mid w_c, w_o)$。

$$
P(D=1\mid w_c, w_o) = \sigma(\boldsymbol{u}_o^\top \boldsymbol{v}_c), \quad  \sigma(x) = \frac{1}{1+\exp(-x)}
$$

其中$\sigma(\boldsymbol{u}_o^\top \boldsymbol{v}_c)$表示skip-gram的输出，表示模型输出的背景词在中心词窗口内部的概率。进一步地，给定一个长度为$T$的文本序列，设时间步$t$的词为$w^{(t)}$，考虑最大化联合概率：

$$
\prod_{t=1}^{T} \prod_{-m \leq j \leq m,\ j \neq 0} P(D=1\mid w^{(t)}, w^{(t+j)})
$$

然而，以上模型中包含的事件仅考虑了正类样本，这样的训练毫无意义，训练得到的模型将无法处理无意义的句子。我们需要通通过添加一些负样本来`人为构建mini-batch`。

$$
\prod_{t=1}^{T} \prod_{-m \leq j \leq m,\ j \neq 0} P(w^{(t+j)} \mid w^{(t)}) 
$$

$P$表示中心词窗口包含背景词的概率，$N_i$表示中心词窗口不包含背景词的概率。

$$
P(w^{(t+j)} \mid w^{(t)}) = P \cdot N_1 \cdot N_2 \cdots N_K = P(D=1\mid w^{(t)}, w^{(t+j)})\prod_{k=1,\ w_k \sim P(w)}^K P(D=0\mid w^{(t)}, w_k)
$$

设文本序列中时间步$t$的词$w^{(t)}$在corpus中的索引为$i_t$，噪声词$w_k$在corpus中的索引为$h_k$，有关以上条件概率的对数损失为：

$$
\begin{split}\begin{aligned}
-\log P(w^{(t+j)} \mid w^{(t)})
=& -\log P(D=1\mid w^{(t)}, w^{(t+j)}) - \sum_{k=1,\ w_k \sim P(w)}^K \log P(D=0\mid w^{(t)}, w_k)\\
=&-  \log\, \sigma\left(\boldsymbol{u}_{i_{t+j}}^\top \boldsymbol{v}_{i_t}\right) - \sum_{k=1,\ w_k \sim P(w)}^K \log\left(1-\sigma\left(\boldsymbol{u}_{h_k}^\top \boldsymbol{v}_{i_t}\right)\right)\\
=&-  \log\, \sigma\left(\boldsymbol{u}_{i_{t+j}}^\top \boldsymbol{v}_{i_t}\right) - \sum_{k=1,\ w_k \sim P(w)}^K \log\sigma\left(-\boldsymbol{u}_{h_k}^\top \boldsymbol{v}_{i_t}\right).
\end{aligned}\end{split}
$$

现在，训练中每一步的梯度计算开销不再与corpus大小相关，而与$K$线性相关。当$K$取较小的常数时，负采样在每一步的梯度计算开销较小。

### Hierarchical softmax
不对这种方法的原理进行深入研究，这部分内容主要介绍其步骤和简单思想。

<center><img src="/img/in-post/word2vec/hierarchical_softmax.pdf" width="60%"></center>

$$
P(w_o \mid w_c) = \prod_{j=1}^{L(w_o)-1} \sigma\left( [\![  n(w_o, j+1) = \text{leftChild}(n(w_o,j)) ]\!] \cdot \boldsymbol{u}_{n(w_o,j)}^\top \boldsymbol{v}_c\right)
$$

其中$n(w_i,j)$表示从根节点到中心词$w_i$叶子节点的路径中第$j$个节点。$\text{leftChild}(n)$表示节点$n$的左子节点。如果$x$为真，则$$[\![ x ]\!] = 1$$，否则$$[\![ x ]\!] = -1$$。

由于$\sigma(x)+\sigma(-x) = 1$性质，因此给定中心词$w_c$生成corpus中任一词的条件概率之和为1这一条件也将满足$\sum_{w \in \mathcal{V}} P(w \mid w_c) = 1$

最后，每次计算梯度反传损失函数数值时只需考虑位于路径中的节点，路径长度$L(w_o)-1$的数量级为$\mathcal{O}(\text{log}_2\mid \mathcal{V}\mid)$，这是hierarchical softmax方法能够大幅度降低bp-loss计算复杂度的原因。

## Reference
> https://zh.d2l.ai/chapter_natural-language-processing/approx-training.html <br>
> https://blog.csdn.net/qq_16438065/article/details/85105829

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内