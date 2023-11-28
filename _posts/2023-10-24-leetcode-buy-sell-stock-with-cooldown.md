---
layout: post
title: "buy and sell stock with cooldown (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-10-24 20:57
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
leetcode 101 - 309 best time to buy and sell stock with cooldown 

<hr>

### # state transition formular (state machine)
```txt
                    ┌───┐
                    │   │
         (-price) ┌─+───┴─┐
           ┌──────┤ idle2 +──────┐
           │      └───────┘      │
        ┌──+──┐               ┌──┴───┐
        │ buy ├───(+prices)───+ sell │
        └──┬──┘               └──+───┘
           │      ┌───────┐      │
           └──────+ idle1 ├──────┘
                  └─+───┬─┘ (+price)
                    │   │
                    └───┘
```

<hr>

### # code
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){ cout << ele << " "; });
    cout << endl;
}

int solve(const vector<int>& vec){
    int n = vec.size();
    if(n<2){                                           // sanity check
        return 0;
    }
    vector<int> buy(n, 0);                             // init vec
    vector<int> idle1(n, 0);
    vector<int> sell(n, 0);
    vector<int> idle2(n, 0);
    buy[0] = idle1[0] = -1*vec[0];                     // init value for the first day
    sell[0] = idle2[0] = 0;
    for(int i=1; i<n; i++){
        buy[i] = idle2[i-1]-vec[i];                    // prev day op could be: idle2 but not sell
        idle1[i] = max(buy[i-1], idle1[i-1]);          // prev day op could be: buy or idle1
        sell[i] = max(idle1[i-1], buy[i-1]) + vec[i];  // prev day op could be: buy or idle1
        idle2[i] = max(sell[i-1], idle2[i-1]);         // prev day op could be: sell or idle2
    }
    return max(sell[n-1], idle2[n-1]);                 // max value is derived after buy or after idle2
}

// dp[i]: the profit value after current op of the (i+1)-th day
int main(){
    vector<int> vec = {1, 2, 3, 0, 2};                 // stock prices each day
    cout << solve(vec) << endl;                        // 3
    return 0;
}
```
