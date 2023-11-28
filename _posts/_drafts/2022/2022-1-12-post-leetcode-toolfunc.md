---
layout: post
title: "leetcode toolfunc"
subtitle: 'leetcode刷题相关的工具函数' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-01-12 15:50
lang: ch 
catalog: true 
categories: leetcode 
tags:
  - Time 2022
---
##### 二叉树的定义
```cpp
// Definition for a binary tree node.
struct TreeNode{
    int val;
    TreeNode *left;
    TreeNode *right;
    TreeNode(): val(0), left(nullptr), right(nullptr){}
    TreeNode(int x): val(x), left(nullptr), right(nullptr){}
    TreeNode(int x, TreeNode *left, TreeNode *right): val(x), left(left), right(right){}
};
```

##### 根据二叉树对应的层序遍历数组构建对应的二叉树
```cpp
// Target: ["1", "3", "null", "null", "2"] => TreeNode* root 
TreeNode* initTree(vector<string> &vec){
    queue<TreeNode*> que;
    TreeNode *root = new TreeNode(atoi(vec[0].c_str()));
    vector<string>::size_type i = 1;
    que.push(root);
    while(!que.empty()){
        int len=que.size();
        while(len--){
            TreeNode *node = que.front();
            que.pop();
            if(vec[i] != "null"){
                node->left = new TreeNode(atoi(vec[i].c_str()));
                que.push(node->left);
            }
            i++;
            if(i == vec.size()){
                return root;
            }
            if(vec[i] != "null"){
                node->right = new TreeNode(atoi(vec[i].c_str()));
                que.push(node->right);
            }
            i++;
            if(i == vec.size()){
                return root;
            }
        }
    }
    return root;
}
```

##### 根据已经构建好的二叉树(TreeNode \*root)按照层序遍历的方式打印对应的vector\<string\>
待整理...
```cpp
vector<vector<string>> newlevelorder(TreeNode *root){
    if(!root){return {};}
    else{
        vector<vector<string>> res;
        queue<TreeNode*> que;
        que.push(root);
        while(!que.empty()){
            vector<string> tmpvec;// hold vec<str> for each level
            int n = que.size();
            int flag = n;
            int mark = 0;
            while(flag--){
                if(que.front()!=nullptr){
                    mark=1; 
                }
                TreeNode *cur=que.front();
                que.pop();
                que.push(cur);
            }
            if(mark!=1){
                break;
            }
            while(n--){
                if(que.front() == nullptr){
                    tmpvec.push_back("null");
                    que.push(nullptr);
                    que.push(nullptr);
                }
                else{
                    tmpvec.push_back(to_string(que.front()->val));
                    que.push(que.front()->left);
                    que.push(que.front()->right);
                }
                que.pop();
            }
            res.push_back(tmpvec);
        }
        return res;
    }
}
```

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
