---
layout: post
title: "partition equal subset (leetcode 101 dynamic programming 01pack)"
author: "melon"
date: 2023-09-12 07:33
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
leetcode 101 - 01 pack problem: 416 partition equal subset

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

bool solve(const vector<int>& vec, int target){
    int n = vec.size();
    vector<vector<bool>> dp(vec.size()+1, vector<bool>(target+1, false));
    dp[0][0] = true;                                           // setup init state
    for(int i=1; i<=vec.size(); i++){                          // first i obj, from vec[0] to vec[i-1]
        for(int j=0; j<=target; j++){                          // each possible subset sum: from 0 to target
            if(j>=vec[i-1]){                                   // the idx=i-1 ele could be added
                dp[i][j] = dp[i-1][j] || dp[i-1][j-vec[i-1]];
            }
            else{
                dp[i][j] = dp[i-1][j];                         // could not be added
            }
            // print2dvec(dp);
        }
    }
    return dp[n][target];
}

// equal partition subset problem: choose or not choose of each item given, to meet the wholesome finish condition
// given sum = accumulate(vec.begin(), vec.end(), 0) -> sum/2 -> choose certain elements to make them addup as sum/2
// dp[i][j]: given first i obj available as 'vec', given a possible sumup value [0-sum/2], the existence of subset 
// inside 'vec' to buildup an equal subset sum.
int main(){
    vector<int> vec = {1, 5, 11, 5};
    int sumup = accumulate(vec.begin(), vec.end(), 0);
    int ans = false;
    if(sumup%2 != 0){
        cout << ans << endl;
    }
    else{
        cout << solve(vec, sumup/2) << endl;
    }
    return 0;
}
```
{% endraw %}

<hr>

### # space compression code (dp[i][j] -> dp[j])
each value of dp[i][j] is only depand on the i-1, so we could compress the 2d dp to 1d, and use inner backward iteration to avoid pollute the "to be used" value.
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <numeric>
#include <iostream>
using namespace std;

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){cout << ele << " ";});
    cout << endl;
}

bool solve(const vector<int>& vec, int target){
    int n = vec.size();
    vector<bool> dp(target+1, false);
    dp[0] = true;                                       // init non
    for(int i=1; i<=n; i++){                            // i: considering first i elements
        for(int j=target; j>=0; j--){                   // j: considering for each possible target sumup value
            if(j>=vec[i-1]){                            // if i-th obj in vec can be added into target sumup
                dp[j] = dp[j] || dp[j-vec[i-1]];        // dont add: use first i-1 obj = previous dp[i] 
            }                                           // add: first i-1 obj + target-value of i-th obj
            else{
                dp[j] = dp[j];                          // i-th obj in vec cannot be added into target sumup
            }
            print1dvec(dp);                             // debug
        }
    }
    return dp[target];
}

int main(){
    // vector<int> vec = {1, 5, 11, 5};                 // 1
    // vector<int> vec = {1, 5, 11, 5, 4, 3};           // 0
    vector<int> vec = {1, 5, 11, 5, 4, 3, 1};           // 1
    int sumup = accumulate(vec.begin(), vec.end(), 0);
    int ans = false;
    if(sumup%2 != 0){
        cout << ans << endl;
    }
    else{
        cout << solve(vec, sumup/2) << endl;
    }
    return 0;
}
```
