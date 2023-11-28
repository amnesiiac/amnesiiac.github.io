---
layout: post
title: "regular expression matching (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-10-16 20:04
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
leetcode 101 - 10 regular expression matching  
need to fully understand the usages of '*' and '.' before conclude the state transition formular.

<hr>

### # code
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

template <typename T>
void print2dvec(const vector<vector<T>>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T ele){ cout << ele << " "; });
        cout << endl;
    }
}

bool solve(const string& str, const string& regex){
    int m = str.length();                                           // string.size() are the same as string.length()
    int n = regex.length();
    vector<vector<bool>> dp(n+1, vector<bool>(m+1, false));
    dp[0][0] = true;                                                // nothing do match with nothing
    for(int j=1; j<n+1; j++){                                       // init dp of "emtpy str" situations
        if(regex[j-1] == '*' && j>=2){                              // any * should appear at least from pos 2
            dp[0][j] = dp[0][j-2];
        };
    }
    for(int i=1; i<m+1; i++){
        for(int j=1; j<n+1; j++){
            if(regex[j-1] == '.'){                                  // regex[j-1] == '.'
                dp[i][j] = dp[i-1][j-1];
            }
            else if(regex[j-1] != '*'){                             // for a normal char
                dp[i][j] = dp[i-1][j-1] && str[i-1]==regex[j-1];    // will match only if str[i-1] == regex[j-1]
            }
            else if(str[i-1]!=regex[j-2] && regex[i-2]!='.' ){      // char regex[j-1] cannot match with str[i-1]
                dp[i][j] = dp[i][j-2];
            }
            else{                                                   // for '*', the ele before * can match with str[i-1]
                dp[i][j] = dp[i][j-1] || dp[i-1][j] || dp[i][j-2];  // * in regex is useless: dp[i][j-1]
            }                                                       // char str[i-1] is useless: dp[i-1][j]
        }                                                           // regex[j-1]* in regex are useless: dp[i][j-2]
        print2dvec(dp);
        cout << endl;
    }
    return dp[m][n];
}

// mainly 3 kinds of characters can be processed: normal char, 
// "*": match 0 or more times of prev char, ".": match exactly 1 time of prev char
// dp[i][j]: given the first i char of str, the first j char of regex, whether they can be "matched"
int main(){
    string str = "aab";
    string regex = "c*a*b";             // 1
    // string regex = "c*ca*b";         // 0
    cout << solve(str, regex) << endl;
}
```
