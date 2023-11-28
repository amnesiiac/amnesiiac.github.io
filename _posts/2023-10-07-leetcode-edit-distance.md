---
layout: post
title: "edit distance (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-10-07 20:31
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
leetcode 101 - 72 edit distance

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <vector>
#include <string>
#include <iostream>
using namespace std;

template <typename T>
void print2dvec(const vector<vector<T>>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T ele){ cout << ele << " "; });
        cout << endl;
    }
}

int solve(string s1, string s2){
    int m = s1.length();
    int n = s2.length();
    vector<vector<int>> dp(m+1, vector<int>(n+1, 0));
    int tmp;
    for(int i=0; i<=m; i++){
        for(int j=0; j<=n; j++){
            if(i == 0){              // no first char, only second char: need j char added
                dp[i][j] = j;
            }
            else if(j == 0){         // no second char, only first char: need i char added
                dp[i][j] = i;
            }
            else{
                // cout << s1.substr(0, i) << "----" << s2.substr(0,j) << endl;  // debug

                // dp[i][j] = min(dp[i-1][j-1]+((s1[i-1] == s2[j-1])? 0:1),      // merged form
                // 		          min(dp[i-1][j]+1, dp[i][j-1]+1));

                if(s1[i-1] == s2[j-1]){                                          // s1[i-1] == s2[j-1]
                    dp[i][j] = dp[i-1][j-1];
                }
                else{                                                            // s1[i-1] != s2[j-1]
                    dp[i][j] = dp[i-1][j-1] + 1;                                 // replace operation
                    dp[i][j] = min(dp[i][j], dp[i-1][j]+1);                      // del s1[i-1], left s2[j-1]
                    dp[i][j] = min(dp[i][j], dp[i][j-1]+1);                      // del s2[j-1], left s1[i-1]
                }
            }
            // print2dvec(dp);                                                   // debug
            // cout << endl;
        }
    }
    return dp[m][n];
}

// dp[i][j]: given the first i char in s1, first j char in s2, the min steps to convert from s1 to s2
// a char conversion including: del char, add char, replace operation
int main(){
    string word1 = "horse";
    string word2 = "ros";
    cout << solve(word1, word2) << endl;
    return 0;
}
```
{% endraw %}
