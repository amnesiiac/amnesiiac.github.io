---
layout: post
title: "leetcode problems summarization"
subtitle: 'leetcode方法相近题目归纳整理' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics\_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-01-19 17:57
lang: ch 
catalog: true 
categories: leetcode 
tags:
  - Time 2022
---
### 一：BFS
##### ➀-➀ recursive BFS(递归BFS遍历方式)
▶︎ **BINARY TREE** <br>

▶︎ **BINARY SEARCH TREE**

▶︎ **GRAPH** 
- **207(course schedule)** [尚未整理] 根据BFS遍历到的图的节点中的indegree和outdegree等特征进行判断。
- **210(course schedule ii)** [尚未整理] 根据BFS遍历到的图的节点中的indegree和outdegree等特征进行判断。


##### ➀-➁ iterative BFS(迭代BFS遍历方式)
▶︎ **GRAPH** 
- **133(clone undirected graph)**: 使用iterative BFS方法(queue数据结构) + unordered_map(对于已经被BFS搜索过的节点进行保存，以减少在使用BFS遍历无向图时可能出现的大量的node重复访问)。

▶︎ **BINARY TREE** 
- **199(binary tree right-sight view)**: 使用iterative BFS方法(queue数据结构)，层次遍历中每行的最后一个遍历到的元素即为应该加入到'right-sight view vec'的节点。
- **200(number of islands)**: 使用iterative BFS方法(queue数据结构)，对于无向图中每个island node，按照宽度优先原则，对于每个island node相邻的四个可能方向上的node加入queue中，并对于每个方向上和初始island node相邻的island node转化成water node，以此区分'孤立'的island node。

▶︎ **BINARY SEARCH TREE** 




### 二：DFS
##### ➁-➀ recursive DFS(递归DFS遍历方式)
▶︎ **BINARY TREE** <br>
在构思recursive DFS遍历算法时，对于二叉树而言，一定需要考虑的是"root node"和"left sub-tree"以及"right sub-tree"之间的关系，而不是纠结于考虑"root node"和"left node"以及"right node"之间的关系。即在考虑设计初始二叉树相关算法时需要考虑到递归形态。
- **199(binary tree right-sight view)**: 使用recursive DFS方法(递归结构：对于当前node的左右子树递归应用DFS，得到当前node的左右子树对应的right-sight view vec，并根据获得两颗子树的right-sight view vec得到整棵树除root节点外的right-sight view vec)，
- **222(count complete tree nodes)**: 使用recursive DFS方法(递归结构：(root_node, root-\>left_subtree, root-\>right_subtree))。进阶解法使用iterative DFS方完成。
- **236(lowest common ancestor)**: 使用recursive DFS方法，目的是构建二叉树中每个node的(node,parent_node)表，以及构建二叉树中每个节点的(node, level)表，便于查询使用。关于最邻近公共先祖节点寻找的算法❮核心思想❯是：首先将较低的那个节点沿着父节点的方向追溯到和较高节点同一level上，然后两个节点同时向上追溯父节点，直到它们遇到第一个公共的父节点即为lowest common ancestor。

▶︎ **BINARY SEARCH TREE**


▶︎ **GRAPH PROBLEMS**
- **133(clone undirected graph)**: 使用recursive DFS方法(递归结构：对于当前node的每个沿深度增加的方向遇到的new_node继续执行DFS) + unordered_map数据结构(对于已经被DFS搜索过的node进行保存，以减少在使用DFS遍历无向图时可能出现的大量的node重复访问)。
- **200(number of islands)**: 使用recursive DFS方法(递归结构：对于每个新发现的island node沿上下左右4个方向执行一次DFS，该DFS的作用是将与新发现的island node相连的island node转变成water node，以此来实现区分'孤立'的island node)。

- **211(add & search word tree)**：首先定义单词查找树(定义其节点数据结构TrieNode)，然后定义查找树添加单词操作(为新添加的单词构建一条从root node到leaf node的路径)，最后在所构建的TrieNode查找树上使用DFS进行单词查找(对于每个TrieNode的leaves，分别执行DFS以验证待搜索单词的后续部分)。
- ➤ **graph circle problems 有向图中环路问题**
    - **207(course schedule)**: 使用recursive DFS方法(递归结构：对于每个graph中的当前start node而言，沿着start node的所有依赖的related node分别进行DFS遍历，对于每个related node再作为starting node，以此类推，进行逐级判断)。这道题目中构造state vector的方法值得学习，在以后的对graph中环路相关的问题可以尝试应用。本题目中，梳理DFS逻辑的关键是"正难则反"，采用"排除法"将一定存在circle的情况剔除，余下的为无环节点。
    - **210(course schedule ii)** 使用recursive DFS方法(递归结构：对于每个graph中的当前start node而言，沿着start node的所有依赖的related node分别进行DFS遍历，对于每个related node再作为starting node，以此类推，进行逐级判断)。这道题目中构造state vector的方法值得学习，在以后的对graph中环路相关的问题可以尝试应用。本题目中，梳理DFS逻辑的关键是"正难则反"，采用"排除法"将一定存在circle的情况剔除，余下的为无环节点，所有无环节点构成答案。

##### ➁-➁ iterative DFS(迭代DFS遍历方式)
▶︎ **BINARY TREE**
- **222(count complete tree nodes)**: 使用iterative DFS方法(基本二叉树遍历：stack\<pair\<TreeNode\*, bool\>\>)。进阶方法利用'complete tree'的性质提升求解时间效率(对于任意一个complete bitree，它的left_sub_tree或right_sub_tree至少有一个是'满二叉树'，而满二叉树的node数量可由height直接计算得到2^height-1，将剩余'非满二叉树'node数量求解交给DFS完成)。

▶︎ **BINARY SEARCH TREE** 
- **230(kth smallest element in BST)**: 使用基础iterative DFS方法，按照中序遍历BST即得到从小到大排列的节点。


▶︎ **GRAPH**
- **133(clone undirected graph)**: 使用iterative DFS方法(采用stack数据结构，对于当前node的每个沿深度增加方向遇到的new_node添加到stack中，以备下次迭代使用) + unordered_map数据结构(对于已经被DFS搜索过的node进行保存，以减少在使用DFS遍历无向图时可能出现的大量的node重复访问)。




## Reference

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用\`\`将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式: <br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色 <br>
> 问题脚注: ### ???problem😫problem???
