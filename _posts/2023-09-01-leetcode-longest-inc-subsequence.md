---
layout: post
title: "longest inc subsequence (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-09-01 22:42
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
leetcode 101 - 300 longest inc subsequence   
as a convension in leetcode: a subsequence need not be continuous, but subarray of substring must be continuous

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int solve(const vector<int>& vec){
    int n = vec.size();
    vector<int> dp(n, 1);
    for(int i=1; i<n; i++){                     // for each pos i
        for(int j=i-1; j>=0; j--){              // for each pos j before i
            if(vec[i] > vec[j]){
                dp[i] = max(dp[i], dp[j]+1);    // find the max inc subsequence for pos i
                print1dvec(dp);
            }
        }
    }
    return *max_element(dp.begin(), dp.end());  // <algorithm.h>: if ret !=dp.end(), the ret is valid
}

// output: the len of longest increasing subsequence
// dp[i]: the longest inc subsequence before vec[i]
// state transition: dp[i] = max(dp[i], dp[i-1])
int main(){
    vector<int> vec = {10, 9, 2, 5, 3, 7, 101, 4};
    print1dvec(vec);
    cout << solve(vec) << endl;
    return 0;
}
```
```txt
10 9 2 5 3 7 101 4
1 1 1 2 1 1 1 1
1 1 1 2 2 1 1 1
1 1 1 2 2 3 1 1
1 1 1 2 2 3 1 1
1 1 1 2 2 3 1 1
1 1 1 2 2 3 4 1
1 1 1 2 2 3 4 1
1 1 1 2 2 3 4 1
1 1 1 2 2 3 4 1
1 1 1 2 2 3 4 1
1 1 1 2 2 3 4 1
1 1 1 2 2 3 4 3
1 1 1 2 2 3 4 3
4
```
{% endraw %}
