---
layout: post
title: "Transformer"
subtitle: 'Attention is all u need'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2020-01-30 21:21 
lang: ch 
catalog: true 
categories: documentation
tags:
  - NLP 
  - attention mechanism
  - Time 2020
---
### Abstract
本文要介绍的是Google提出的针对提升机器翻译(NMT-Neural Machine Translation)任务效果的论文[Attention is all you need](https://arxiv.org/abs/1706.03762)。这篇文章是在阅读原始论文以及很大程度借鉴了[这篇博文](https://jalammar.github.io/illustrated-transformer/)中的图片、解析思路并且经过本人的反复综合分析之后补充整理的结果。

### Overview 
如下图所示，整个NMT任务接受一种语言作为输入，经过翻译模块`transformer`之后输出目标语言。
<center><img src="/img/in-post/transformer/overview0.pdf" width="80%"></center>

其中`transformer`结构可以进一步分解成主要两个部分:`encoders`，`decoders`，详见下图。
<center><img src="/img/in-post/transformer/overview1.pdf" width="80%"></center>

其中`encoders`部分以及`decoders`部分分别由6个子组件构成，每个子组件结构相同，但是不共享参数，通过首尾相连构成。
<center><img src="/img/in-post/transformer/overview2.pdf" width="80%"></center>

##### Encoder
每个`encoder`由`self-attention`和`feed-forward network`两部分组成。
<center><img src="/img/in-post/transformer/encoder.pdf" width="60%"></center>

引用博客中的原文对上述结构进行描述:
> `self-attention`: a layer that helps the encoder look at other words in the input sentence as it encodes a specific word. <br>
> `feed-forward network`: The outputs of the self-attention layer are fed to a feed-forward neural network. The exact same feed-forward network is independently applied to each position.

##### Decoder
每个`decoder`由`self-attention`和`encoder-decoder attention`以及`feed-forward network`三个部分组成。注意到，解码器除了具有编码器的两种结构，还具有一个特殊的attention结构层，该层的作用是：使得解码器在解码的过程中能够考虑当前单词周围相关的词汇信息。
<center><img src="/img/in-post/transformer/decoder.pdf" width="40%"></center>

<center><img src="/img/in-post/transformer/en_de_coder.pdf" width="80%"></center>

### Details 
这部分内容针对overview中描述的框架中的细节进行描述。

##### Encoder
**word embedding**  对于输入的句子中的每个单词，通过`word embedding`的方式将所有单词编码成相同长度的向量(论文中为512)。
<center><img src="/img/in-post/transformer/embedding.pdf" width="80%"></center>

**embeddings into encoder**  下图展示了单词向量输入到`encoder`中的过程，每个`encoder`的都会接收一个nested python list(list中的每个元素都是512维的词向量)，list的第一维尺度是可以设置的超参，通常选择训练集中的最长句子包含的单词个数作为list的长度。如前一小节图示，前一级`encoder`的输出是后一级`encoder`的输入。
<center><img src="/img/in-post/transformer/embedding_encoder.pdf" width="60%"></center>

另外，上图中的python list中的每个词向量的在`self-attention`中的参数路径是独立的，准确的说，不同的词向量经过的是一个`multi-head attention`。但是，所有位于一个python list中的词向量经过模型中`feed-forward network`结构时，共享相同的网络参数结构。

**encoder into encoder**
下图展示了不同`encoder`之间的关系。
<center><img src="/img/in-post/transformer/encoder_encoder.pdf" width="60%"></center>

##### Self-attention #1
假设我们需要翻译的输入句子为：**The animal didn’t cross the street because it was too tired**，在人进行翻译的时候需要通过语言逻辑确定句中**it**所指代的事物，但是如何通过算法编码使得其能够自动将**it**和**The animal**联系起来呢?`self-attention`就可以实现这个功能，如下图所示，`self-attention`模块的建模结果已经用连接线的颜色轻重表示**it**和**The animal**之间的联系紧密程度。<br>

<center><img src="/img/in-post/transformer/self_attention.pdf" width="40%"></center>

下面对其原理进行解读：当模型处理单词**it**时，`self-attention`允许将**it**和**animal**联系起来。当模型处理每个位置的词时，`self-attention`允许模型看到句子的其他位置信息作辅助线索来更好地编码当前词。RNN中也可以看到实现类似功能结构：RNN的隐状态传递允许了之前的词向量来解释合成当前词的解释向量，而`Transformer`使用`self-attention`来将相关词的理解编码到当前词中。

**❮Self attention feed forward in details❯** <br>
**➤ step-1 生成三个向量** <br>
对于编码器输入的每个词向量，生成$query$、$key$、$value$矩阵，可以简化的将它们分别称作$Q$、$K$、$V$矩阵。$Q$、$K$、$V$矩阵的生成方法如下图所示，编码器输入向量$X_i$(512维)分别乘以三个转化矩阵$W_Q$、$W_K$、$W_V$，这些矩阵参数在训练过程中需要学习。<br>
需要注意的是，第一：并不是每个输入的词向量都分别拥有独立的转化矩阵，而是所有输入词向量共享相同的转换矩阵；第二：$W_Q$、$W_K$、$W_V$转化矩阵是基于输入位置$pos_i$的转换矩阵，输入位置$pos_i$个数一般取训练数据集中一句话中包含的最多的单词数。

<center><img src="/img/in-post/transformer/step1.pdf" width="60%"></center>

一般情况下，编码器输入向量(512)经过$Q$、$K$、$V$转化矩阵生成新向量的维度比输入词向量的维度要小(64)，实验发现这种设计有助于`multi-head attention`算法收敛。

**➤ step-2**<br>
以$Thinking \ Machines$为例，为$Thinking$($pos_1$)计算`self-attention`数值，即为$pos_1$的单词计算和当前输入句子中其他所有位置$pos_i$单词的分数，这个分数决定了编码器对$Thinking$进行编码时，句中其他词对于编码当前词附加的关联度。其计算步骤如下图所示。

<center><img src="/img/in-post/transformer/step2.pdf" width="60%"></center>

**➤ step-3 && step-4** <br>
将*➤ step-2*中得到的关联分数除以$dimension_{key}$(可以使得更新梯度更加稳定)，然后再经过$softmax$处理成和为1的形式：$softmax(\frac{score}{\sqrt{dimension_{key}}})$，其操作流程见下图：

<center><img src="/img/in-post/transformer/step34.pdf" width="60%"></center>

**➤ step-5**<br>
将softmax操作得到的当前单词的`self-attention`相关程度分数与当前单词的$Value$矩阵按位相乘(即每个单词的softmax分数分别和其$Value$矩阵中的每个元素相乘)，用于保留关联性大的单词的$Value$值，削弱相关性差的单词的$Value$值。

**➤ step-6**<br>
将所有当前单词$Thinking$产生的加权向量$v_1$、$v_2$进行按元素加和，得到当前单词$Thinking$的`self-attention`模块输出结果$z_1$，同理，针对当前输入句子中其他单词$Machines$采用上述6个步骤可以得到输出向量$z_2$。

<center><img src="/img/in-post/transformer/step56.pdf" width="60%"></center>

**❮Self attention feed forward in Matrix form❯** <br>
可以使用矩阵形式完成上述6个步骤以加速算法。

**➤ step-1**

<center><img src="/img/in-post/transformer/self_attention_matrix0.pdf" width="80%"></center>

**➤ step-2**

<center><img src="/img/in-post/transformer/self_attention_matrix1.pdf" width="60%"></center>

其中，$Q$矩阵和$K$矩阵的元算结果相当于对于输入句子中的每个单词计算其`self-attention map`，`map`中的分数表示单词之间相互关联程度。
<center><img src="/img/in-post/transformer/self_attention_map.pdf" width="40%"></center>

**❮Multi-head self attention -- ❮Ensemble learning❯❯** <br>
`Transformer`论文中增加了`multi-head`机制用于增强`self-attention`结构的能力，其增强效果主要在于下面两个方面：<br>
> 一：`multi-head`机制扩展了模型将注意力集中于不同位置的能力，这种设计可以看作是一种变种的`ensemble learning`。在上面的例子中，$z_1$只包含了其他词的很少信息❮12%❯，$z_1$向量很大程度上❮88%❯由当前单词决定。在其他情况下，比如翻译**The animal didn’t cross the street because it was too tired**时，我们想知道单词**it**指的是什么，在这种场景下，`single-head`很难满足要求。 <br>
> 二：`multi-head`机制赋予attention多种表达方式，论文中使用`8-head`机制相当于对于输入自注意力模式能够展示8种分型表达方式。如下图所示，每一组参数矩阵都经过随机初始化，经过训练之后，输入向量可以被映射到8种不同的子表达空间中。

**➤ step-1** 每个`head`都具有互相独立的一组$W_Q$、$W_K$、$W_V$参数转化矩阵，和输入相乘得到$Q$、$K$、$V$矩阵。
<center><img src="/img/in-post/transformer/multihead0.pdf" width="40%"></center>

**➤ step-2** 每个`head`得到的$Q$、$K$、$V$矩阵分别进行处理得到8个`self-attention`矩阵❮$z_0$--$z_7$❯。
<center><img src="/img/in-post/transformer/multihead1.pdf" width="40%"></center>

**➤ step-3** 将上一步骤得到的$z_0$到$z_7$先进行`concatenate`操作，然后和随机初始化的参数转化矩阵$W_0$相乘，得到`multi-head self-attention`的结果矩阵$z$。
<center><img src="/img/in-post/transformer/multihead2.pdf" width="60%"></center>

**➤ multi-head self-attention overview**<br>
单个`encoder`内部结构矩阵运算形式示意图如下所示。
<center><img src="/img/in-post/transformer/multihead_overview.pdf" width="80%"></center>

需要注意的是，对于`#encoder0`而言，在输入之前需要进行2次`embedding`处理❮一次词向量`embedding`，一次位置信息嵌入`embedding`❯，但是，对于`#encoder0`后面级联的`#encoder1...`不需要此操作。

**➤ multi-head visualization** <br>
用不同颜色代表不同的`head`，查看`self-attention`内部权重关联程度如下图所示，图中以橙色代表`#head0`，以绿色代表`#head1`。为了能够清晰显示单词之间的依赖关系，其他的`#headi`没有绘出。

<center><img src="/img/in-post/transformer/multihead_visualization.pdf" width="40%"></center>

根据图中描述，可做如下判断：编码器在编码**it**时，`#head0`集中于**the animal**，另一个`#head1`集中于**tired**，某种意义上讲，模型对**it**的表达借助了输入句子中**animal**和**tired**两个单词。

**➤ Representing The Order of The Sequence Using Positional Encoding** <br>
为了实现机器翻译，不仅要让模型理解每个单词对应的意思，还需要让其能对于每个单词应处于的正确位置具有编解码能力。这一部分用于介绍`Transformer`模型编码单词位置相关信息的方式。

**➤ step-1** `word-embedding`和`pos-embedding`在`Transformer`模型中的编码方式概览。

<center><img src="/img/in-post/transformer/positionencoding0.pdf" width="60%"></center>

**➤ step-2** 最长单词数量为20个，每个单词经过`word-embedding`得到的结果为512维向量所需要的位置信息编码表。表中左右两个部分每个位置的颜色数值分别由$sin$和$cos$函数确定，编码函数选取方式不唯一，但是要求编码方法必须能够处理未知长度的序列(即对任意数量单词输入不同行之间`数值pattern`保持唯一性)。表中每一行都和相应位置的`word-embedding`结果相加得到`pos-embedding`结果向量。

<center><img src="/img/in-post/transformer/positionencoding2.pdf" width="60%"></center>

**➤ step-3** 以`word-embedding`结果是一个4维向量为例，使用额外4维`pos-embedding`向量进行位置信息嵌入。

<center><img src="/img/in-post/transformer/positionencoding1.pdf" width="60%"></center>

**❮The Residuals❯** <br>
这部分内容介绍`Transformer`模型中`Residual—connnection`结构，内容引用自[这篇博客](https://jalammar.github.io/illustrated-transformer/)原文。
> One detail in the architecture of the encoder that we need to mention before moving on, is that each sub-layer (`self-attention`, `FFNN`) in each encoder has a residual connection around it, and is followed by a `layer-normalization` step.

<center><img src="/img/in-post/transformer/residual0.pdf" width="60%"></center>

> If we’re to visualize the vectors and the layer-norm operation associated with self attention, it would look like this:

<center><img src="/img/in-post/transformer/residual1.pdf" width="40%"></center>

> This goes for the sub-layers of the decoder as well. If we’re to think of a Transformer of 2 stacked encoders and decoders, it would look something like this:

<center><img src="/img/in-post/transformer/residual2.pdf" width="60%"></center>

##### Decoder
**❮Decoder structure overview❯** <br>
介绍完`encoders`部分，下面介绍`decoders`部分。首先通过两个gif来了解`encoders`和`decoders`协同工作原理。
首先位于最后面的`encoder`的输出被转换为$K$矩阵和$V$矩阵，这两个矩阵被每个`decoder`中的`encoder-decoder attention`层来使用，帮助`decoder`集中注意力于输入序列的合适位置。解码器`decoder`的输入序列不仅包括来自`encoders`的输出矩阵$K$和$V$，还包括来自训练集中每个句子的目标语言`label`经过`word-embedding+pos-embedding`后的序列。下面的gif是`Transformer`模型单步运行的流程。

<center><img src="/img/in-post/transformer/decoder0.gif" width="80%"></center>

下面的gif展示了`Transformer`模型翻译一句话的整个流程，相当于上面的gif中的步骤一直重复直到一个特殊符号❮end of sequence❯出现表示解码器完成了翻译输出。`encoder`之间互相级联，前一个`encoder`的输出被送入下一个`encoder`中。训练集中的每个句子中的标签❮目标语言的翻译句子经过`word-embedding`以及`pos-embedding`❯作为第一个`decoder`中的输入。

<center><img src="/img/in-post/transformer/decoder1.gif" width="80%"></center>

##### Self-attention #2
**❮Multi-head attention和Masked Multi-head attention的区别❯** <br>
`encoders`中的`self-attention`是`Multi-head attention`，`decoders`中的`self-attention`是`Masked Multi-head attention`。其区别如下图所示：

<center><img src="/img/in-post/transformer/self_attention_differ.pdf" width="60%"></center>

其中灰色部分使用`0-mask`或者`-inf-mask`进行遮罩，使得翻译当前单词的时候不能够看到未来将要输入的单词。具体地，**I**作为第一个单词，只能有和**I**自己的attention。**am**作为第二个单词，具有**i**，**am**两个单词的attention。**a**作为第三个单词，具有**i**，**am**，**a**前面三个单词的attention。最后一个单词**student**，才具有对整个句子4个单词的attention。

另外，解码器中的`encoder decoder attention`结构和编码器中的`encoder attention`结构相同，不同之处在于，`decoder`中的$Q$矩阵来自前面一层，$K$、$V$矩阵从`encoder`中获得。
> The `Encoder-Decoder Attention` layer works just like multiheaded self-attention, except it creates its Queries matrix from the layer below it, and takes the Keys and Values matrix from the output of the encoder stack.

<center><img src="/img/in-post/transformer/self_attention_masked.pdf" width="60%"></center>

##### Linear-softmax layer
`decoders`之后用于词汇翻译结构如下图所示。
<center><img src="/img/in-post/transformer/linear_softmax.pdf" width="60%"></center>

### Training
**❮Word Encoding❯** 下面图像中展示了单词的`one-hot`编码过程，经过编码的单词可以通过和`label`计算`cross entropy`或`KL-divergence`来进一步计算误差函数。
<center><img src="/img/in-post/transformer/output_encode2.pdf" width="80%"></center>

**❮Target Model Outputs❯** 下面图像中展示了针对完整的一句话的`trainging label`。`one-hot`编码数量一般为训练集中`vocab_size`。
<center><img src="/img/in-post/transformer/output_target.pdf" width="40%"></center>

**❮Trained Model Outputs❯** 下面图像中展示了`transformer`模型经过训练得到的正确的`logit`输出。
<center><img src="/img/in-post/transformer/output_logit.pdf" width="40%"></center>

**❮Model Decode Methods❯** <br>
**1** `greedy search`: 一般情况下，模型一次`decode`只产生一个输出单词。这时，会使用`argmax`函数，将当前模型对于当前位置`#position i`中所有单词`logits`输出最大对应的单词选定作为当前位置的输出，训练集词库中其他单词不予考虑。<br>
**2** `beam search`: 另外一种情况下，模型一次`decode`中产生`beam_size`个输出单词。假设`beam_size=2`，此时对于当前位置`#position i`中所有单词`logits`输出中的`top-2`对应的两个单词选定作为当前位置的`候选单词`(假设是`I`和`a`)，在下一个位置`#position i+1`中，分别以`#position i`位置的单词是`I`或者`a`，运行模型两次，最终$$whichever\ version\ $$$$produced\ less\ $$$$error\ considering\ $$$$both\ positions\ $$$$i\ and\ i+1\ $$$$is\ kept.$$

### Code
```python
# todo
```

## Reference
> https://jalammar.github.io/illustrated-transformer/ <br>
> [Łukasz Kaiser’s talk](https://www.youtube.com/watch?v=rBCqOTEfxvg) <br>
> 

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号 <br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号 <br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
