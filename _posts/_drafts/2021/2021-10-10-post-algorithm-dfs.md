---
layout: post
title: "deep first search (DFS)"
subtitle: '[数据结构与算法] - 深度优先搜索算法' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-10-11 09:28
lang: ch 
catalog: true 
categories: algorithm 
tags:
  - Time 2021
---
### 一：引言
深度优先搜索是一种图搜索算法之一，

之所以称为深度优先搜索算法是因为：搜索算法总是在已发现节点的基础上寻找未发现节点，而深度优先搜索在搜索下一个未发现节点时总是按照"深度"优先"的方式查找。在给定源节点s以及沿着s的一条出发边到达的节点v的情况下，深度优先搜索算法总是优先搜索v的出发边，而不是返回s以寻找和v深度相同的其他节点；对于每次搜索，总是按照深度优先的原则进行寻找，直到图中所有节点都被发现为止，最终生成由多棵深度优先树构成的深度优先森林。

### 二：DFS在一个简单有向图上的推演过程
<center><img src="/img/in-post/algorithm_img/dfs_1.pdf" width="100%"></center>

### 三：DFS推演过程伪代码
```cpp
DFS(G)  // G canbe directed graph or undirected graph
    for each vertex u in G.V  // init each node in graph
        u.color = YELLOW
        u.pre = NULL
    time = 0  // set init traverse time
    for each vertex u  in G.V // for each un-traversed node, execute DFS_VISIT
        if u.color == YELLOW
            DFS_VISIT(G,u)
DFS_VISIT(G,u)  // BFS search core func
    time = time + 1  // set current temp-source node time
    u.firstvalue = time
    u.color = GRAY  // set node state: searched
    // recursive call DFS_VISIT for each attached node v of u
    for each v in G_Adj(u)  
        if v.color == YELLOW
            v.pre = u  //  set pre-node
            DFS_VISIT(G,v)
    u.color = BLACK  // set u's all sub nodes search finished
    time = time + 1
    u.secondvalue = time  // set u second search order value
```


### 四：DFS搜索的性质
**性质#1** 深度优先搜索提供了关于图的价值较高的信息：DFS遍历生成的每一个前驱子图$G\_{pre}$都是一棵深度优先搜索树，整个遍历的结果得到一个深度优先搜索森林。保证DFS搜索结果能够生成深度优先搜索树的条件是：DFS\_VISIT函数的节点调用顺序和深度优先搜索树的结构完全对应；即DFS\_VISIT(G,v$只在对u节点的邻接链表(G\_Adj(u))搜索中调用，即u节点是v节点的唯一前驱pre。

**性质#2** 使用深度优先搜索遍历图时，每个节点的发现时间和完成时间具有所谓的"括号化结构(parenthesis structure)"，参考本文第二小节中的图可以更好的理解。发现时间是指节点从"未遍历"的状态转变成"初次遍历"的状态。


### 五：DFS在特殊图(二叉树)的遍历中的应用
二叉树的前序、中序、后序遍历都属于图的深度优先搜索。3种遍历方式的图解和代码如下。

##### 1 二叉树的前序遍历图解(根->左子树->右子树)
<center><img src="/img/in-post/algorithm_img/dfs_2.pdf" width="100%"></center>
##### 1 二叉树的前序遍历c++代码
```cpp
// recursive traverse #1
vector<int> preorderTraversal(TreeNode* root){
    vector<int> result;
    if(!root){return result;}
    else{
        result.push_back(root->val);// root
        for(auto itleft:preorderTraversal(root->left)){// left subtree
            result.push_back(itleft->val);
        }
        for(auto itright:preorderTraversal(root->right)){// right subtree
            result.push_back(itright->val);
        }
        return result;
    }
}
```
```cpp
// recursive traverse #2
vector<int> preorderTreaversal(TreeNode* root){
    vector<int> result;
    if(!root){return result;}
    else{
        result.push_back(root->val);// root
        result.insert(result.end(),
            preorderTraversal(root->left).begin(), 
            preorderTraversal(root->left).end());
        result.insert(result.end(), 
            preorderTraversal(root->right).begin(), 
            preorderTraversal(root->right).end());
        return result;
    }
}
```
```cpp
// no-recursive method => stack
vector<int> preorderTraversal(TreeNode* root){
    vector<int> result;
    if(!root){return result;}
    else{
        bool visited;
        stack<pair<TreeNode*, bool>> stk;// core data structure
        stk.push(make_pair(root, false));// init
        while(!stk.empty()){
            root=stk.top().first;
            visited=stk.top().second;
            stk.pop();
            if(!root){// handle traversal boundary
                continue;// if root=nullptr => noop
            }
            if(visited){// if visited => add to the result vec
                result.push_back(root->val);
            } 
            else{// preorder -> root->left->right
                stk.push(make_pair(root->right, false));
                stk.push(make_pair(root->left, false));
                stk.push(make_pair(root, true));
            }
        }
        return result;
    }
}
```


##### 2 二叉树的中序遍历图解(左子树->根->右子树)
<center><img src="/img/in-post/algorithm_img/dfs_3.pdf" width="100%"></center>
##### 2 二叉树的中序遍历c++代码
```cpp
// recursive traverse method
vector<int> inorderTraversal(TreeNode* root){
    vector<int> result;
    if(!root){return result;}
    else{
        for(auto itleft:inorderTraversal(root->left)){// left subtree
            result.push_back(itleft->val);
        }
        result.push_back(root->val);// root
        for(auto itright:inorderTraversal(root->right)){// right subtree
            result.push_back(itright->val);
        }
        return result;
    }
}
```
```cpp
// non-recursive traverse method
vector<int> inorderTraversal(TreeNode* root){
    vector<int> result;
    if(!root){return result;}
    else{
        bool visited;
        stack<pair<TreeNode*, bool>> stk;
        stk.push(make_pair(root, false));
        while(!stk.empty()){
            root = stk.top().first; 
            visited = stk.top().second;
            stk.pop();
            if(!root){
                continue;
            }
            if(visited){
                result.push_back(root->val);
            }
            else{// left->root->right
                stk.push(make_pair(root->right, false));
                stk.push(make_pair(root, true));
                stk.push(make_pair(root->left, false));
            }
        }
        return result;
    }
}
```


##### 3 二叉树的后序遍历图解(左子树->右子树->根)
<center><img src="/img/in-post/algorithm_img/dfs_4.pdf" width="100%"></center>
##### 3 二叉树的后序遍历c++代码
```cpp
// recursive traverse method
vector<int> postorderTraversal(TreeNode* root){
    vector<int> result;
    if(!root){return result;}
    else{
        for(auto itleft : postorderTraversal(root->left)){// left subtree
            result.push_back(itleft->val);
        }
        for(auto itright : postorderTraversal(root->right)){// right subtree
            result.push_back(itright->val);
        }
        result.push_back(root->val);// root
    }
}
```
```cpp
// non-recursive traverse method
vector<int> postorderTraversal(TreeNode* root){
    vector<int> result;
    if(!root){return result;}
    else{
        bool visited;
        stack<pair<TreeNode*, bool>> stk;
        stk.push(make_pair(root, false));
        while(!stk.empty()){
            root = stk.top().first;
            visitied = stk.top().second;
            if(!root){
                continue;
            }
            if(visited){
                result.push_back(root->val);
            }
            else{// left->right->root
                stk.push(make_pair(root, true));
                stk.push(make_pair(root->right, false));
                stk.push(make_pair(root->left, false));
            }
        }
        return result;
    }
}
```

## Reference
> 算法导论 3rd part-6 chapter-22.3

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
