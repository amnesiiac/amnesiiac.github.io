---
layout: post
title: "best time to buy and sell stock (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-10-09 20:21
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
leetcode 101 - 121 best time to buy and sell stock

<hr>

### # code
{% raw %}
method-1: using normal method which has O(n^2) time complexity.
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
    int maxval = INT_MIN;
    for(int i=0; i<vec.size()-1; i++){                       // the time of buy
        for(int j=i+1; j<vec.size(); j++){                   // the time of sell
            maxval = max(maxval, vec[j]-vec[i]);             // find the max buy-sell
            cout << i << " " << j << " " << maxval << endl;  // debug
        }
    }
    return maxval;
}

int main(){
    vector<int> vec = {7, 1, 5, 3, 6, 4};
    cout << solve(vec) << endl;                              // 5
    return 0;
}
```
method-2: a simplified method which has O(n^1) time complexity.
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
    int profit = 0; 
    int cost = INT_MAX;
    for(int i=0; i<vec.size(); i++){
        cost = min(cost, vec[i]);               // min cost
        profit = max(profit, vec[i]-cost);      // max profit
        cout << cost << " " << profit << endl;
    }
    return profit;
}

int main(){
    vector<int> vec = {7, 1, 5, 3, 6, 4};
    cout << solve(vec) << endl;                 // 5
    return 0;
}
```
debug info:
```text
cost profit
7    0
1    0
1    4
1    4
1    5
1    5
5
```
{% endraw %}
