---
layout: post
title: "word break (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-08-31 21:56
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
leetcode 101 - 139 word break 

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;


int solve(const string s, vector<string>& wordset){
    int n = s.length();                                  // total n chars
    vector<bool> dp(n+1, false);                         // dp[0] is for init, dp[1]~dp[n] is for string
    dp[0] = true;
    int len;
    for(int i=1; i<=n; i++){
        for(string word : wordset){
            len = word.length();
            if(i>=len && s.substr(i-len, len) == word){  // if idx i is a possible pos && the segments exist in wordset
                dp[i] = dp[i] || dp[i-len];              // dp[i]: dp[i-len] || dp[i] already true in pre search
            }
        }
        print1dvec(dp);
    }
    return dp[n];
}

// dp[i]: whether the string can be separated using word in word at the i pos of the string
int main(){
    string s = "applepenapplepenpenapplecat";
    vector<string> wordset = {"apple", "pen", "cat"};
    cout << solve(s, wordset) << endl;
    return 0;
}
```
{% endraw %}
