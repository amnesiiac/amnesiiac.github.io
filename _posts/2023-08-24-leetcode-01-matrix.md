---
layout: post
title: "01 matrix (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-08-24 21:09
categories: "2023"
tags:
  - leetcode
---

### # dynamic programming algorithm
(1) construct "state transition metric".  
(2) make sure the boundary condition that could startup the "transition".  
(3) the state transition metric value stores the userful result to solve the problem.  
(4) the startup condition is the boundary condition.

<hr>

### # theme description
leetcode 101 - 542 01-matrix

<hr>

### # redundancy version
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

void solve(vector<vector<int>>& vec, vector<vector<int>>& dp){
    int m = vec.size();
    int n = vec[0].size();
    if(m<=0 || n<=0){
        return;
    }
    // first iteration
    for(int i=0; i<m; i++){                                  // search rightbottom -> lefttop 
        for(int j=0; j<n; j++){
            if(i==0 && j==0){                                // no left + no top
                if(vec[0][0]==0){
                    dp[0][0] = 0;
                }
                continue;
            }
            if(i==0 && j>0){                                 // no top + has left 
                if(vec[0][j]==0){
                    dp[0][j]=0;
                }
                else{
                    dp[0][j] = min(dp[0][j-1]+1, dp[0][j]);
                }
                continue;
            }
            if(i>0 && j==0){                                 // has top + no left
                if(vec[i][0]==0){
                    dp[i][0]=0;
                }
                else{
                    dp[i][0] = min(dp[i-1][0]+1, dp[i][0]);
                }
                continue;
            }
            if(vec[i][j]==0){                                // has top + has left
                dp[i][j] = 0;
            }
            else{
                dp[i][j] = min(dp[i-1][j], dp[i][j-1])+1;
            }
        }
    }
    // second iteration: use the dp derived above
    for(int i=m-1; i>=0; i--){                               // search lefttop -> rightbottom
        for(int j=n-1; j>=0; j--){
            if(i==m-1 && j==n-1){                            // no right + no bottom
                if(vec[m-1][n-1]==0){
                    dp[m-1][n-1] = 0;
                }
                continue;
            }
            if(i==m-1 && j<n-1){                             // has right + no bottom
                if(vec[m-1][j]==0){
                    dp[m-1][j] = 0;
                }
                else{
                    dp[m-1][j] = min(dp[m-1][j], dp[m-1][j+1]+1);
                }
                continue;
            }
            if(i<m-1 && j==n-1){                             // no right + has bottom
                if(vec[i][n-1]==0){
                    dp[i][n-1] = 0;
                }
                else{
                    dp[i][n-1] = min(dp[i][n-1], dp[i+1][n-1]+1);
                }
                continue;
            }
            if(vec[i][j]==0){                                // has bottom + has right
                dp[i][j] = 0;
            }
            else{
                dp[i][j] = min(dp[i][j], min(dp[i][j+1], dp[i+1][j])+1);
            }
        }
    }
}

// dp[i][j]: the shortest distance from current pos (i,j) to 0
// state transition: dp[i][j] = min(dp[i-1][j], dp[i+1][j], dp[i][j-1], dp[i][j+1])+1 if(vec[i][j] !=0) else 0

// problem(thorny): the above state transition formula cannot inference because the dp[i][j+1] and dp[i+1][j] is not present

// method-1: separate the movement into 2 parts: one from lefttop to rightbottom, another from rightbottom to lefttop 
// method-2: enable bfs search with metrics to memorize the searched pos
int main(){
    vector<vector<int>> vec = {{0, 0, 0}, {0, 1, 0}, {1, 1, 1}};
    vector<vector<int>> dp(vec.size(), vector<int>(vec[0].size(), INT_MAX));
    print2dvec(vec);
    solve(vec, dp);
    print2dvec(dp);
    return 0;
}
```

<hr>

### # simplified version (combine if clauses)
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

void solve(vector<vector<int>>& vec, vector<vector<int>>& dp){
    int m = vec.size();
    int n = vec[0].size();
    if(m<=0 || n<=0){
        return;
    }
    for(int i=0; i<m; i++){                                 // search rightbottom -> lefttop 
        for(int j=0; j<n; j++){
            if(vec[i][j]==0){                               // no need to search outside 
                dp[i][j] = 0;
                continue;
            }
            if(j>0){                                        // has left 
                dp[i][j] = min(dp[i][j-1]+1, dp[i][j]);
            }
            if(i>0){                                        // has top
                dp[i][j] = min(dp[i-1][j]+1, dp[i][j]);
            }
        }
    }
    for(int i=m-1; i>=0; i--){                              // search lefttop -> rightbottom
        for(int j=n-1; j>=0; j--){
            if(vec[i][j]==0){                               // no need to search outside
                dp[i][j] = 0;
                continue;
            }
            if(j<n-1){                                      // has right
                dp[i][j] = min(dp[i][j], dp[i][j+1]+1);
            }
            if(i<m-1){                                      // has bottom
                dp[i][j] = min(dp[i][j], dp[i+1][j]+1);
            }
        }
    }
}

int main(){
    vector<vector<int>> vec = {{0, 0, 0}, {0, 1, 0}, {1, 1, 1}};
    vector<vector<int>> dp(vec.size(), vector<int>(vec[0].size(), INT_MAX));
    print2dvec(vec);
    solve(vec, dp);
    print2dvec(dp);
    return 0;
}

```
```tet
0 0 0
0 1 0
1 1 1

0 0 0
0 1 0
1 2 1
```

<hr>

### # method-3: using bfs to search 4 directions with cache to avoid rep
todo
{% endraw %}
