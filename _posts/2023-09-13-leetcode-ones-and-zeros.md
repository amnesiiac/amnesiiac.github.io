---
layout: post
title: "ones and zeros (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-09-13 22:00
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
leetcode 101 - 474 ones and zeros 

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <iostream>
#include <string>
#include <vector>
#include <utility>
using namespace std;

template <typename T>
void print2dvec(const vector<vector<T>>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T& ele){cout << ele << " ";});
        cout << endl;
    }
    cout << endl;
}

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){cout << ele << " ";});
    cout << endl;
}

pair<int, int> count(const string& str){                        // return num of 0/1 of a given str
    int count0 = str.length();
    int count1 = 0;
    for(char c : str){
        if(c == '1'){
            count1++;
            count0--;
        }
    }
    return make_pair(count0, count1);
}

int solve(const vector<string>& vec, int ones, int zeros){
    int idx = vec.size();
    int count0, count1;
    vector<vector<vector<int>>> dp(idx+1, vector<vector<int>>(ones+1, vector<int>(zeros+1, 0)));
    int oldk;
    for(int k=1; k<=idx; k++){                                  // for first k in vec as "target arr"  vec[k]
        // auto [count0, count1] = count(vec[k]);
        pair<int, int> ret = count(vec[k-1]);
        count0 = ret.first;
        count1 = ret.second;
        cout << "---" << vec[k-1] << "---" << endl;
        for(int i=1; i<=ones; i++){                             // for each num of 1
            for(int j=1; j<=zeros; j++){                        // for each num of 0
                if(i>=count1 && j>=count0){                     // if current num of 1/0s can buildup the str
                    dp[k][i][j] = max(dp[k-1][i][j], 1+dp[k-1][i-count1][j-count0]);    // build or not build
                }
                else{
                    dp[k][i][j] = dp[k-1][i][j];
                }
                print2dvec(dp[k]);
            }
        }
    }
    return dp[idx][ones][zeros];
}

// dp[i][j][k]: the max num of string that using i*1 + j*0 to buildup the first k string of vec 
int main(){
    vector<string> vec = {"10", "0001", "111001", "1", "0"};
    int m = 3;                         // 1
    int n = 5;                         // 0
    cout << solve(vec, m, n) << endl;
    return 0;
}
```

<hr>

### # space compression: from dp[k][i][j] to dp[i][j]
just reduce dimension k, and reverse inner 2 layer traversal order
{% endraw %}
