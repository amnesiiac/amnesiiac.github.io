---
layout: post
title: "coin change (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-09-27 19:31
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
leetcode 101 - 322 coin change (complete pack problem)

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

template <typename T>
void print2dvec(const vector<vector<T>>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T& ele){cout << ele << " ";});
        cout << endl;
    }
    cout << endl;
}

int solve(const vector<int>& vec, int sumup){
    int n = vec.size();
    if(n<=0){
        return -1;
    }
    vector<vector<int>> dp(n+1, vector<int>(sumup+1, sumup+1));
    dp[0][0] = 0;                                                  // init startup value (nonsence)
    for(int i=1; i<=n; i++){                                       // first i coin available 
        for(int j=0; j<=sumup; j++){                               // to meet the sumup as j [0,sumup]
            if(vec[i-1]<=j){                                       // if cur coin can be count in
                dp[i][j] = min(dp[i-1][j], dp[i][j-vec[i-1]]+1);   // let in or not in
            }
            else{
                dp[i][j] = dp[i-1][j];                             // cur coin cannot be count in 
            }
            print2dvec(dp);                                        // debug
        }
    }
    return dp[n][sumup];
}

// dp[i][j]: using first i ele inside vec(idx:[0,i-1]), the min num of coin to addup as j[0,sumup]
int main(){
    vector<int> vec = {1, 2, 5};
    int amount = 11;
    cout << solve(vec, amount) << endl;  // output the minimal number of coins used to meet the given sumup 
    return 0;
}
```

<hr>

### # space compression from 2d to 1d
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <iostream>
using namespace std;

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){cout << ele << " ";});
    cout << endl;
}

int solve(const vector<int>& vec, int sumup){
    int n = vec.size();
    if(n<=0){
        return -1;
    }
    vector<int> dp(sumup+1, sumup+1);                  // init value for min algorithm
    dp[0] = 0;                                         // init startup value
    for(int i=1; i<=n; i++){                           // first i coin available 
        for(int j=0; j<=sumup; j++){                   // to meet the sumup as j (mind the traversal order)
            if(vec[i-1]<=j){                           // if cur coin can be count in
                dp[j] = min(dp[j], dp[j-vec[i-1]]+1);  // let in or not in
            }
            else{
                dp[j] = dp[j];                         // cur coin cannot be count in 
            }
            print1dvec(dp);                            // debug
        }
    }
    return dp[sumup];
}

int main(){
    vector<int> vec = {1, 2, 5};
    int amount = 11;
    cout << solve(vec, amount) << endl;
    return 0;
}
```
{% endraw %}
