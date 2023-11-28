---
layout: post
title: "islands distance (leetcode 101 dfs to init + dfs que to solve)"
author: "melon"
date: 2023-08-17 21:30
categories: "2023"
tags:
  - leetcode
---

### # bfs algorithm

<hr>

### # theme description
leetcode 101 - 934 shortest bridge

<hr>

### # code
{% raw %}
utils:
```text
#ifndef UTILS_H_
#define UTILS_H_

#include <vector>
#include <queue>
#include <iostream>
using namespace std;

template<typename T>
void print2dvec(vector<vector<T>>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T ele){cout << ele << " ";});
        cout << endl;
    }
}

#endif
```
main source:
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}

#include "utils.h"

// separate two by marking island=2
void dfs(queue<pair<int, int>>& que, vector<vector<int>>& vec, int i, int j, bool& searched){
    if(i<0 || i>=vec.size() || j<0 || j>=vec[0].size()){             // if invalid pos
        return;
    }
    if(vec[i][j]==0 || vec[i][j]==2){                                // if no need to search or already searched
        return;
    }
    if(vec[i][j]==1){
        vec[i][j] = 2;
        que.push(make_pair(i, j));
        searched = true;
    }
    dfs(que, vec, i-1, j, searched);
    dfs(que, vec, i+1, j, searched);
    dfs(que, vec, i, j-1, searched);
    dfs(que, vec, i, j+1, searched);
}

int solve(vector<vector<int>>& vec){
    vector<int> directions = {-1, 0, 1, 0, -1};
    // find the first islands
    int m = vec.size();
    int n = vec[0].size();
    bool searched = false;
    queue<pair<int, int>> que;                                       // bfs queue (init)
    for(int i=0; i<m; i++){
        for(int j=0; j<m; j++){
            if(searched){
                goto compute;                                        // jump out of multiple for loop
            }
            dfs(que, vec, i, j, searched);                           // dfs search to mark all
        }
    }
compute:
    int x, y;
    int level = 0;
    while(!que.empty()){                                             // bfs
        int npoints = que.size();
        level++;
        while(npoints--){                                            // deal with all step 1 from start points
            // int r = que.front().first; 
            // int c = que.front().second; 
            auto [r, c] = que.front();
            que.pop();
            for(int d=0; d<4; d++){                                  // compute 4 directions
                x = r + directions[d];
                y = c + directions[d+1];
                cout << "current dealing with: " << x << ", " << y << endl;
                if(x<0 || x>=vec.size() || y<0 || y>=vec[0].size()){ // if invalid pos
                    continue;
                }
                if(vec[x][y] == 2){                                  // if not target island
                    continue;
                }
                if(vec[x][y] == 1){                                  // if meet target island
                    return level;
                }
                if(vec[x][y]==0){
                    que.push(make_pair(x, y));
                    vec[x][y] = 2;
                    print2dvec(vec);
                }
            }
        }
    }
    return 0;
}

int main(){
    vector<vector<int>> vec={{1, 1, 1, 1, 1}, {1, 0, 0, 0, 1}, {1, 0, 1, 0, 1}, {1, 0, 0, 0, 1}, {1, 1, 1, 1, 1}};
    // vector<vector<int>> vec={{1, 1, 1, 1, 1}, {0, 0, 0, 0, 0}, {0, 0, 0, 0, 0}, {0, 0, 0, 0, 0}, {1, 1, 1, 1, 1}};
    cout << solve(vec)-1 << endl;
    return 0;
}
```
output:
```txt
1
```

{% endraw %}
