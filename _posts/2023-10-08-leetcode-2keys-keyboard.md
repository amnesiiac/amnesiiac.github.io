---
layout: post
title: "2 keys keyboard (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-10-08 22:08
categories: "2023"
tags:
  - leetcode
---

### # dynamic programming algorithm
(1) construct "state transition metric".  
(2) make sure the boundary condition that could startup the "transition".  
(3) the state transition metric value stores the userful result to solve the problem. Analysis the state transition graphs, firstly determine the type of operations can meet the requirement, then find out the "start state" matches each conversion operation, finally the wholesome state transition metric is derived.  
(4) the startup condition is the boundary condition.

<hr>

### # theme description
leetcode 101 - 650 2 keys keyboard

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){ cout << ele << " "; });
    cout << endl;
}

int solve(const int& num){
    vector<int> dp(num+1, 0);             // dp
    for(int i=1; i<=num; i++){            // dp[i] is made up of dp[j]
        dp[i] = i;                        // init the min step to form AAA need num of paste(max)
        for(int j=2; j<num; j++){         // mind the range of j
            if(i%j==0){
                dp[i] = dp[j] + dp[i/j];
                break;
            }
        }
        print1dvec(dp);
    }
    return dp[num];
}

// charater operations: copy, paste the prev copy content
// dp[i]: give a startup char A, the min steps to generated AAAAAA..AAA (totally i A)
// the particularity of cur problem: the state transition formular is made up of it own prev values
int main(){
    int num = 4;
    cout << solve(num) << endl;
    return 0;
}
```
```txt
0 1 0 0 0
0 1 3 0 0
0 1 3 4 0
0 1 3 4 6
6
```
{% endraw %}
