---
layout: post
title: "graph basics"
subtitle: '[数据结构与算法] - 图的基本表示' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-10-11 09:58
lang: ch 
catalog: true 
categories: algorithm 
tags:
  - Time 2021
---
### 一：引言
关于图主要有两部分内容：**图的表示**和**图的搜索**。

图表示顾名思义，就是按照一定数据结构将图的节点和边的关系表示出来；图搜索是指：系统化地跟随图的边来访问图中每个节点。

进一步地，对于未知但存在的一个图而言，图搜索算法可以用于发现图结构，许多基于图的高级算法在初始化阶段都会事先通过图搜索算法来获取图结构，并使用一定图表示方式进行记录。图搜索技术是整个图算法领域的核心内容。

图表示方式是图论中的基础内容，本文主要对于两种图表示方式进行介绍。


### 二：图的表示
将图记为G=(V,E)，其中V表示图的节点，E表示图的边。对于图G=(V,E)来说，可以用两种标准方式进行表示，一种表示方式是将G=(V,E)表示成**邻接链表的组合**，另一种表示方式是将G=(V,E)看作是一个**邻接矩阵**。

两种表示方式都可以表示无向图、有向图。<br>
其中邻接链表的组合方式在表示稀疏图时，表示形式更加紧凑而成为更加通常的选择。稀疏图是指边的数量($\mid E\mid$)远远小于节点数量的平方($\mid V^2\mid$)的图；在表示图的时候，表示的复杂度和边的数量以及节点数量的平方成线性相关。<br>
在图为稠密图时，即当图的边数量($\mid E\mid$)和图中节点数量($\mid V^2\mid$)相近时，邻接矩阵通常是更好的选择。<br>
另外，如果整个算法中需要快速判断给定两个节点之间是否有边连接，则大概率需要使用邻接矩阵表示法。

##### 1 使用邻接链表表示有向图、无向图、有权图、无权图
**(i) 无向图G=(V,E)的邻接链表表示**
<center><img src="/img/in-post/algorithm_img/graph_1.pdf" width="80%"></center>

**(ii) 有向图G=(V,E)的邻接链表表示**
<center><img src="/img/in-post/algorithm_img/graph_2.pdf" width="80%"></center>

**(iii) 无向有权重图G=(V,E)的邻接链表表示**
<center><img src="/img/in-post/algorithm_img/graph_3.pdf" width="100%"></center>
- 对于有权图中两个节点之间不存在边的情况，可以在相应行列记录中存放值NIL。但是对于实际问题而言，可以根据问题实际数值意义，用0或者∞或者其他数值作为问题的"罚项"，从而表示该节点对之间不存在边。

**(iv) 有向有权图G=(V,E)的邻接链表表示**
<center><img src="/img/in-post/algorithm_img/graph_4.pdf" width="100%"></center>
- 对于有权图中两个节点之间不存在边的情况，可以在相应行列记录中存放值NIL。但是对于实际问题而言，可以根据问题实际数值意义，用0或者∞或者其他数值作为问题的"罚项"，从而表示该节点对之间不存在边。

**邻接链表表示法的潜在缺陷**<br>
使用邻接链表表示的图的形式无法快速判断一条边(u,v)是否是图中的边，唯一的方式是：对无向图而言，在邻接链表Adj[u]中搜索节点v，或者在邻接链表Adj[v]中搜索节点u；对于有向图而言，只能在邻接链表Adj[u]中搜索节点v。

##### 2 使用邻接矩阵表示有向图、无向图、有权图、无权图
**(i) 无向图G=(V,E)的邻接矩阵表示**
<center><img src="/img/in-post/algorithm_img/graph_5.pdf" width="80%"></center>
- 无向图邻接矩阵是一个对称矩阵，即邻接矩阵$A=A^T$。
- 在某些应用中，存储无向图的邻接矩阵时，为了节省图存储空间，只存储邻接矩阵对角线及其上面的半部分。

**(ii) 有向图G=(V,E)的邻接矩阵表示**
<center><img src="/img/in-post/algorithm_img/graph_6.pdf" width="100%"></center>
- 有向图的邻接矩阵表示没有对称性，即(u,v)和(v,u)两个节点分别对应不同的边。有向图中每条边都有方向。

**(iii) 无向有权图G=(V,E)的邻接矩阵表示**
<center><img src="/img/in-post/algorithm_img/graph_7.pdf" width="90%"></center>
- 对于有权图中两个节点之间不存在边的情况，可以在相应行列记录中存放值NIL。但是对于实际问题而言，可以根据问题实际数值意义，用0或者∞或者其他数值作为问题的"罚项"，从而表示该节点对之间不存在边。

**(iv) 有向有权图G=(V,E)的邻接矩阵表示**
<center><img src="/img/in-post/algorithm_img/graph_8.pdf" width="100%"></center>
- 对于有权图中两个节点之间不存在边的情况，可以在相应行列记录中存放值NIL。但是对于实际问题而言，可以根据问题实际数值意义，用0或者∞或者其他数值作为问题的"罚项"，从而表示该节点对之间不存在边。

##### 3 图中节点、边的属性的表示方式
大多数算法都需要对建立的图结构中的节点、边的属性进行维护，这些属性可以通过如下方式进行表示：节点v的属性d可以表示为v.d；边(u,v)的属性f可以表示为(u,v).f。

上述方式在文字表述图中节点、边的属性时，足够清晰，但是在具体编程实现图相关算法时，还可能需要设计相应数据结构类型来实现图带有属性的节点、边这两种数据结构。


## Reference
> 算法导论 3rd part-6 chapter-22.1

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用\`\`将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式：<br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色 <br>
> 11 使用html设置可折叠部分内容：<br>
  `<details>` <br>
      `<summary><b>[点击展开] xxx</b></summary>` <br>
      `<center><img src="/img/in-post/tcp-ip_img/tcp_20_9.pdf" width="100%"></center>` <br>
  `</details>` <br>
> 12 问题脚注: ???problem😫problem???
