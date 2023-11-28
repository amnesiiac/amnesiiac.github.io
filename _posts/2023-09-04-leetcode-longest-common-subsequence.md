---
layout: post
title: "longest common subsequence (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-09-04 21:19
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
leetcode 101 - 1143 longest common subsequence   
as a convension in leetcode: a subsequence need not be continuous, but subarray of substring must be continuous

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <string>
#include <vector>
#include <iostream>
using namespace std;

int solve(const string& s1, const string& s2){
    int m = s1.length();
    int n = s2.length();
    vector<vector<int>> dp(m+1, vector<int>(n+1, 0));         // init value at (0,0), not ele in string
    for(int i=1; i<=m; i++){
        for(int j=1; j<=n; j++){
            if(s1[i-1] == s2[j-1]){                           // if pos i,j ele in s1,s2 are equal
                dp[i][j] = dp[i-1][j-1]+1;                    // num = pos i-2,j-2 (dp[i-1][j-1]+1)
            }
            else{                                             // if pos i,j in s1,s2 are not equal
                dp[i][j] = max(dp[i-1][j]+1, dp[i][j-1]+1);   // num = max(fallback of either s1/s2)
            }
        }
    }
    return dp[m][n];
}

// output: the len of longest common subsequence
// dp[i][j]: the longest common subsequence, at pos i,j of s1,s2 (index i-1, j-1)
// state transition: dp[i][j] = dp[i-1][j-1]+1 if s1[i-1]=s2[j-1]
//                   dp[i][j] = max(dp[i][j-1], dp[i][j-1]) else
int main(){
    // string s1 = "abc";
    string s1 = "oxcpqrsvwf";
    // string s1 = "abecd";
    // string s2 = "def"; 
    string s2 = "shmtulqrypy"; 
    cout << solve(s1, s2) << endl;
    return 0;
}
```
```txt
2
```
{% endraw %}
