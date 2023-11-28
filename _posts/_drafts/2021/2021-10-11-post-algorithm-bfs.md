---
layout: post
title: "breadth first search (BFS)"
subtitle: '[数据结构与算法] - 广度优先搜索' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-10-11 22:37
lang: ch 
catalog: true 
categories: algorithm 
tags:
  - Time 2021
---
### 一：引言
广度优先搜索是最简单的图搜索算法之一，也是许多重要的图算法的原型：prim最小生成树算法和dijkstra单源最短路径算法都用到了bfs的基本思想。

给定一个图G=(V,E)，和一个可识别的源节点s。广度优先搜索能够沿着图的边按照"一定规则"对图中的节点进行探索，从而发现源节点s能够到达的所有节点，并且能够获取从源节点到每个可达节点的距离(经过最少的边数)，最终生成一棵"广度优先搜索树"。

之所以称为广度优先搜索算法是因为：搜索算法总是在已发现节点的基础上寻找未发现节点，而广度优先搜索在搜索下一个未发现节点时总是按照"广度"优先"的方式查找，即该算法在搜索距离源节点s距离为k的所有节点之后，才去搜索距离源节点s为k+1的节点。


### 二：BFS算法在一个简单无向图上的推演过程
<center><img src="/img/in-post/algorithm_img/bfs_1.pdf" width="100%"></center>

### 三：BFS推演过程伪代码
```cpp
BFS(G,s)
// init point in G except s
for each vertex u in G.V - {s}
    // property#1 color of each node, indicates whether the node's in queue
    u.color = YELLOW  
    // property#2 distance of each node 
    u.value = infinity  
    // node u's antecedent node 
    u.pre = NULL
// init source node s
s.color = GRAY
s.value = 0
s.pre = NULL
// init que
que = NULL
ENQUE(que,s)
// BFS searching ...
while(que!=NULL)
    u = DEQUE(que)  // derive current node for handling
    for each v in G.Adj[u]  // for each attached node of u
        if v.color == YELLOW  // if node havent been traverse
            v.color = GRAY  // set node enqued
            v.value = v.value + 1  // distance++
            v.pre = u  // set node pre
            ENQUE(v)// enque u's child node 
    u.color = BLACK // set dequed node u's color
```


### 四：使用BFS算法寻找给定图中两个节点之间的最短路径
广度优先搜索能够找到图G=(V,E)中任一给定源节点s到所有可以到达的节点之间的距离。定义从源节点s到达节点v中所有路径中，历经的最少的边的数量即为$(s,v)$之间的最短路径距离$\delta (s,v)$；如果$(s,v)$之间没有路径，则记$\delta (s,v)=\infty$。

关于证明BFS算法能够正确计算出图中任意两个可达节点的最短路径距离，可以参考\<算法导论\> 3rd p361 最短路径部分的内容。


### 五：BFS算法在对图进行搜索时将创建一颗广度优先搜索树
BFS在对图G=(V,E)搜索的过程中，所生成的**前驱子图**是一棵广度优先搜索树。前驱子图是原图G=(V,E)中的一部分，它包含原子图所有的节点，以及以部分边。

关于证明BFS的遍历图的过程生成的前驱子图是一棵广度优先搜索树的证明，详见\<算法导论 3rd part6 chapter22.2 p363 引理22.6\>


### 六：BFS算法在特殊图(二叉树)的遍历中的应用
##### 1 二叉树的层序遍历图解
<center><img src="/img/in-post/algorithm_img/bfs_2.pdf" width="80%"></center>

##### 2 二叉树的层序遍历c++代码
```cpp
// recursive traverse method
vector<vector<int>> levelorder(TreeNode* root){
    // construct an 2d vec with each subvec hold node of each level
    vector<vector<int>> result;
    // core recursive func
    toolfunc(root, 0, result);
    return result;
}

// root(level) -> left(level+1) -> right(level+1)
void toolfunc(TreeNode* node, int level, vector<vector<int>>& vec){ 
    if(!node) return;
    if(vec.size() == level){// if need enlarge subvec number in vec
        vec.push_back({});
    } 
    vec[level].push_back(node->val);// deal with root
    if(node->left){// deal with left
        levelorder(node->left, level+1, vec);
    }
    if(node->right){// deal with right
        levelorder(node->right, level+1, vec);
    }
}
```
```cpp
// non-recursive traverse method
vector<vector<int>> levelorder(TreeNode* root){
    vector<vector<int>> result;
    if(!root){return {};}
    else{
        queue<TreeNode*> que;// core data structure: queue
        que.push(root);// root
        while(!que.empty()){
            vector<int> tmpvec;
            // infinite loop: for(int i=0; i<que.size(); i++)
            for(int i=que.size(); i>0; i--){
                TreeNode* node=que.front();
                tmpvec.push_back(node->val);
                que.pop();
                if(node->left){// left
                    que.push(node->left);
                }
                if(node->right){// right
                    que.push(node->right);
                }
            }
            result.push_back(tmpvec);
        }
        return result;
    }
}
```

## Reference
> 算法导论 3rd part-6 chapter-22.2

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
