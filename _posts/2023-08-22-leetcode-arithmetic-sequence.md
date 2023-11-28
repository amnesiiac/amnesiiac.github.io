---
layout: post
title: "arithmetic sequence (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-08-22 22:15
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
leetcode 101 - 413 arithmetic sequence 

<hr>

### # original dp version with space complicity of O(n^2)
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int solve(vector<vector<int>>& vec){
    if(vec.size()<=0 || vec[0].size()<=0){
        return 0;
    }
    vector<vector<int>> dp(vec.size(), vector<int>(vec[0].size(), 0));  // init 2d dp
    for(int i=0; i<vec.size(); i++){
        for(int j=0; j<vec[0].size(); j++){
            if(i==0 && j==0){                                           // first ele
                dp[0][0] = vec[0][0];
                continue;
            }
            if(i==0 && j>0){                                            // no above
                dp[i][j] = dp[0][j-1] + vec[i][j];
                continue;
            }
            if(j==0 && i>0){                                            // no left
                dp[i][j] = dp[i-1][0] + vec[i][j];
                continue;
            }
            dp[i][j] = min(vec[i][j]+dp[i-1][j], vec[i][j]+dp[i][j-1]);
        }
    }
    return dp.back().back();                                            // return last of last ele
}


// dp[i][j]: min path at the pos (i,j)
// state transition: dp[i][j] = min(vec[i][j]+dp[i-1][j], vec[i][j]+dp[i][j-1]);
int main(){
    vector<vector<int>> vec = {{1, 3, 1}, {1, 5, 1}, {4, 2, 1}};
    print2dvec(vec);
    cout << solve(vec) << endl;
    return 0;
}
```

<hr>

### # improved dp with space complicity of O(n)
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

// squeeze the space takeup of dp
// when computing each dp[i][j], only dp[i-1][j] or dp[i][j-1] is used,
// and the values before the j-th col will never be used anymore
// thus, we could use a dp(vec[0].size(), 0) to hold te necessary states
int solve(vector<vector<int>>& vec){
    if(vec.size()<=0 || vec[0].size()<=0){
        return 0;
    }
    vector<int> dp(vec[0].size(), 0);                            // init 1d dp
    for(int i=0; i<vec.size(); i++){
        for(int j=0; j<vec[0].size(); j++){
            if(i==0 && j==0){                                    // first ele
                dp[0] = vec[0][0];
            }
            if(i==0 && j>0){                                     // first row ele
                dp[j] = dp[j-1] + vec[0][j];
            }
            if(i>0 && j==0){                                     // first col ele
                dp[0] = dp[0] + vec[i][0];
            }
            dp[j] = min(dp[j]+vec[i][j], dp[j-1]+vec[i][j]);     // the newer dp[j] is due to older dp[j](upper)
        }                                                        // or the dp[j-1](former)
    }
    return dp.back();                                            // return last ele
}

// dp[i][j]: min path at the pos (i,j)
// state transition: dp[i][j] = min(vec[i][j]+dp[i-1][j], vec[i][j]+dp[i][j-1]);
int main(){
    vector<vector<int>> vec = {{1, 3, 1}, {1, 5, 1}, {4, 2, 1}};
    print2dvec(vec);
    cout << solve(vec) << endl;
    return 0;
}
```
{% endraw %}
