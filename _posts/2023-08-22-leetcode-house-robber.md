---
layout: post
title: "house robber (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-08-22 22:11
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
leetcode 101 - 198 house robber

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int solve(vector<int>& house){
    int numhouse = house.size()-1;
    vector<int> dp(numhouse, 0);                 // alloc dp vector
    dp[0] = house[0];                            // setup startup condition
    dp[1] = max(house[0], house[1]);
    for(int i=2; i<=numhouse; i++){
        dp[i] = max(dp[i-1], dp[i-2]+house[i]);  // the max robber when reach i = max[rob(i-1)+no rob(i), rob(i-2)+rob(i)]
    }
    return dp[numhouse];
}

// robber cannot rob two adjacent house 
// dp[i]: max cash when robber reaches the (i-1)-th house 
// dp[i] = max(dp[i-1], nums[i]+dp[i-2])  i>=2
// i=1: return nums[1]; i=0 return 0
int main(){
    vector<int> house = {2, 7, 9, 3, 1};
    cout << solve(house) << endl;
    return 0;
}
```
{% endraw %}
