---
layout: post
title: "perfect squares (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-08-27 11:05
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
leetcode 101 - 279 perfect squares

<hr>

{% raw %}
```text
#include <vector>
#include <iostream>
#include "utils.h"

using namespace std;

int solve(int num){
    vector<int> dp(num, INT_MAX);
    dp[0] = 0;                    // number 0 is zero number of perfect square
    for(int i=1; i<=num; i++){
        for(int j=1; j*j<num; j++){
            dp[i] = min(dp[i], dp[i-j*j])+1;  // dp[i] always store the min num of perfect square
        }
    }
    return dp[num];
}

// prerequisite: each number can be separated into a series of perfect squares 
// dp[i]]: min num of perfect squrares that buildup number i
// state transition: dp[i] = min(dp[i-1], dp[i-4], dp[i-9]...)+1
// the way to derive the state transition func: just 1 step back
int main(){
    int num = 13;
    cout << solve(num) << endl;
    return 0;
}
```

{% endraw %}
