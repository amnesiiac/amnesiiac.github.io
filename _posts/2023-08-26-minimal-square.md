---
layout: post
title: "minimal square (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-08-26 17:37
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
leetcode 101 - 221 minimal square 

<hr>

{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

template <typename T>
void print2dvec(const vector<vector<T>>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T ele){cout << ele << " "; });
        cout << endl;
    }
    cout << endl;
}

// get min max value from a 2d vector
template <typename T>
vector<T> edgeval(const vector<vector<T>>& vec){
    T minans = INT_MAX;
    T maxans = INT_MIN;
    vector<T> ans;
    for(auto& row: vec){
        for(auto val: row){
            minans = min(minans, val);
            maxans = max(maxans, val);
        }
    }
    return {minans, maxans};
}

int solve(vector<vector<int>>& vec){
    int m=vec.size(); int n=vec[0].size();
    if(m<=0 || n<=0){                                     // skip condition
        return -1;
    }
    vector<vector<int>> dp(m, vector<int>(n, INT_MAX));   // init dp
    for(int i=0; i<m; i++){
        for(int j=0; j<n; j++){
            if(vec[i][j]==0){                             // if no need to search outside, set radius 0
                dp[i][j] = 0;
                continue;
            }
            dp[i][j] = 1;                                 // vec[i][j]!=0, set radius 1
            if(i>1){                                      // cur pos has top
                dp[i][j] = max(dp[i][j], dp[i-1][j]+1);
            }
            if(j>1){                                      // cur pos has left
                dp[i][j] = max(dp[i][j], dp[i][j-1]+1);
            }
            if(i>1 && j>1){                               // cur pos has left + top
                dp[i][j] = max(dp[i][j], dp[i-1][j-1]+1);
            }
        }
    }
    print2dvec(dp);
    return edgeval(dp)[1];
}

// dp[i][j]: the square radius which right-bottom located at pos(i,j) 
// if vec[i][j]=0 -> dp[i][j] = 0 
// if vec[i][j]=1 -> dp[i][j] = k if dp[i-1][j] && dp[i][j-1] && dp[i-1][j-1] >=k-1
// 1 1 1 1 0
// 1 1 1 1 0
// 1 1 1 1 1
// 1 1 1 1 1 (4)(2)
// 0 0 1 1 1 (2)(3)
int main(){
    vector<vector<int>> vec = {{1,0,1,0,0}, {1,0,1,1,1}, {1,1,1,1,1},{1,0,0,1,0}};
    print2dvec(vec);
    cout << solve(vec) << endl;
    return 0;
}
```
output:
```txt
1 0 1 0 0
1 0 1 1 1
1 1 1 1 1
1 0 0 1 0

1 0 1 0 0
1 0 1 2 3
2 1 2 3 4
3 0 0 4 0

4
```

{% endraw %}
