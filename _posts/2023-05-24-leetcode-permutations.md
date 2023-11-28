---
layout: post
title: "permutations (leetcode 101 backtracking)"
author: "twistfatezz"
date: 2023-05-24 22:42
categories: "2023"
tags:
  - leetcode
---

### # backtracking algorithm
todo

<hr>

### # theme description
Given an array of integers without repeating numbers, find all their permutations.  
The algorithm can be implemented as:  
```txt
swap 0 with 0:  [1,2,3]  ->  level 0
           dfs: [1,2,3]  ->  level 1
           dfs: [1,3,2]  ->  level 2
swap 0 with 1:  [2,1,3]  ->  level 1
           dfs: [2,3,1]  ->  level 2
swap 0 with 2:  [3,2,1]  ->  level 2
```

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <stack>
#include <utility>
#include <iostream>

using namespace std;

void dfs(vector<int> &vec, int level, vector<vector<int>> &res){
    if(level==vec.size()-1){                    // cannot swap and seach any more
        res.push_back(vec);
        return;
    }
    for(int i=level; i<vec.size(); i++){
        swap(vec[i], vec[level]);               // swap i with level
        dfs(vec, level+1, res);                 // dfs search results starting after swap
        swap(vec[i], vec[level]);               // revert the change, go on next session
    }
}

vector<vector<int>> permute(vector<int> &vec){
    vector<vector<int>> res;
    dfs(vec, 0, res);                           // dfs entrypoint
    return res;
}

int main(){
    vector<int> vec={1, 2, 3};
    vector<vector<int>> res = permute(vec);
    for(auto v:res){
        for(auto i:v){
            std::cout << i << " "; 
        }
        std::cout << ", ";
    }
    return 0;
}
```
the permutations of vector 1, 2, 3 is:
```txt
1 2 3 , 1 3 2 , 2 1 3 , 2 3 1 , 3 2 1 , 3 1 2 ,
```

{% endraw %}
