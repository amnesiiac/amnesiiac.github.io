---
layout: post
title: "climb stairs (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-08-21 22:25
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
leetcode 101 - 70 climb stairs 

<hr>

### # simplified version to using O(1) 'dp'
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int solve(int num){
    if(num == 1 || num == 2){
        return num;
    }
    int prev = 0; int next = 1; int cur = -1;
    for(int i=2; i<=num; i++){
        cur = prev + next;
        prev = next;
        next = cur;
        cout << cur << " " << next << " " << prev << endl;
    }
    return cur;
}

// note: this method not covering "large number lead to int overflow" problem
// i: num of total stairs
// dp[i]: method to reach i stairs by 1 step or 2 step
// state transition function: dp[i] = dp[i-1] + dp[i-2] (i>=2)
// margin condition: i (i<2)
int main(){
    int numstairs = 5;
    cout << solve(numstairs) << endl;
    return 0;
}
```
{% endraw %}
