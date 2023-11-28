---
layout: post
title: "connective to pacific/atlantic (leetcode 101 dfs stack)"
author: "twistfatezz"
date: 2023-05-23 17:53
categories: "2023"
tags:
  - leetcode
---

### # theme description
Given a two-dimensional non-negative integer matrix, the value of each position represents the altitude.
Assuming that the left and upper sides are the Pacific Ocean, and the right and lower sides are the Atlantic Ocean, find out from which positions water can flow down to the Pacific Ocean and the Atlantic Ocean.
Water can only flow from a location at a higher altitude to a location at a lower elevation or the same.
```txt
pacific ~ ~ ~ ~ ~ ~/*
      ~ 1 2 2 3 5 *
      ~ 3 2 3 4 4 *
      ~ 2 4 5 3 1 *
      ~ 6 7 1 4 5 *
      ~ 5 1 1 2 4 *
    ~/* * * * * * * atlantic
```
<hr>

### # connective to pacific/atlantic both (dfs) - stack version
{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

void dfs(const vector<vector<int>> &vec, vector<vector<int>> &res, const int start){
    vector<int> direct = {-1, 0, 1, 0, -1};
    for(int i=0; i<vec.size(); i++){
        for(int j=0; j<vec[0].size(); j++){
            if(i==start || j==start){ // dfs start point: pacific/atlantic coastline
                if(res[i][j]==0){ // if not searched yet
                    res[i][j]=1; 
                    stack<pair<int, int>> stk; // hold dfs
                    stk.push(make_pair(i, j));
                    int row, col;
                    while(!stk.empty()){
                        // auto coord = stk.pop();
                        pair<int, int> coord = stk.top();
                        stk.pop();
                        for(int d=0; d<4; d++){
                            // coordinate of start point
                            row = coord.first+direct[d];
                            col = coord.second+direct[d+1];
                            if(row>=0 && row<vec.size() && col>=0 && col<vec[0].size()){ // check
                                // if not searched yet, current position can flow to last one
                                if(!res[row][col] && vec[row][col]>=vec[coord.first][coord.second]){
                                    stk.push(make_pair(row, col));  // add to dfs path stack
                                    res[row][col]=1; // add to result
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

vector<vector<int>> vec_and(const vector<vector<int>> &v1, const vector<vector<int>> &v2){
    vector<vector<int>> result(v1.size(), vector<int>(v1[0].size()));
    // vector<vector<int>> result;
    for(int i=0; i<v1.size(); i++){
        for(int j=0; j<v1[0].size(); j++){
            result[i][j] = v1[i][j] && v2[i][j];
            // if(v1[i][j] && v2[i][j]){
            //     result.push_back({i, j});
            // }
        }
    }
    return result;
}

void print_vec(const vector<vector<int>> &vec){
    for(auto v:vec){
        for(auto i:v){
            std::cout << i << " ";
        }
        std::cout << std::endl;
    }
}

vector<vector<int>> connective_both_pacific_atlantic(vector<vector<int>> &vec){
    vector<vector<int>> pacific_connective(vec.size(), vector<int>(vec[0].size(), 0));
    dfs(vec, pacific_connective, 0);
    // print_vec(pacific_connective);
    vector<vector<int>> atlantic_connective(vec.size(), vector<int>(vec[0].size(), 0));
    dfs(vec, atlantic_connective, vec.size()-1);
    // print_vec(atlantic_connective);
    return vec_and(pacific_connective, atlantic_connective);
}

int main(){
    vector<vector<int>> vec={{1,2,2,3,5}, {3,2,3,4,4}, {2,4,5,3,1}, {6,7,1,4,5}, {5,1,1,2,4}};
    auto res = connective_both_pacific_atlantic(vec);
    print_vec(res);
    return 0;
}
```
test samples:
```txt
1 2 2 3 5     1 4 2 2 1
3 2 3 4 4     3 2 3 1 6
2 4 5 3 1     5 4 5 2 1
6 7 1 4 5     6 7 1 4 5
5 1 1 2 4     5 1 1 2 4
```
test result:
```txt
0 0 0 0 1     0 1 1 1 1
0 0 0 1 1     0 0 1 0 1
0 0 1 0 0     0 0 1 0 0
1 1 0 0 0     1 1 0 0 0
1 0 0 0 0     1 0 0 0 0
```

<hr>

### # connective to pacific/atlantic both (dfs) - recursive version
```cpp
pending
```
```txt
pending
```
{% endraw %}
