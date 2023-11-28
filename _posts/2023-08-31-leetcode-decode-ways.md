---
layout: post
title: "decode ways (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-08-31 07:27
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
leetcode 101 - 91 decode ways 

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int solve(string s){
    int n=s.length();                                     // return num of chars in s
    if(n==0){return 0;}
    int prev = s[0] - '0';
    if(prev == 0){                                        // if s == "0" --> x/v
        return 0;
    }
    if(n==1){                                             // if only 1 char(0-9), and not equal 0
        return 1;
    }
    vector<int> dp(n+1, 1);                               // for s len >=2
    int next=-1;
    for(int i=2; i<n+1; i++){                             // for each possible end pos i=index+1
        next = s[i-1] - '0';                              // set next
        if(next==0 && (prev==0 || prev>2)){               // xx
            return 0;
        }
        if((prev==1 || prev==2) && (next>0 && next<7)){   // vv v/v
            dp[i] = dp[i-1] + dp[i-2];
        }
        if((prev>0 && prev<3) && (next==0)){              // vv
            dp[i] = dp[i-2];
        }
        if((prev==2 && next>6) || (prev>2 && next>0)){    // v/v
            dp[i] = dp[i-1];
        }
        print1dvec(dp);                                   // debug
        prev = next;                                      // set prev
    }
    return dp[n];
}

// dp[i]: all possible decode ways from the substr (0,i)
// state transition: the function varies by servral conditions
// given a decode group xy
//    x   0    1       2       (2,7)   >=7
// y
// 0      xx   vv      vv      xx      xx
// 1/2    x/v  vv v/v  vv v/v  v/v     v/v
// >3,<7  x/v  vv v/v  vv v/v  v/v     v/v
// >=7    x/v  vv v/v  v/v     v/v     v/v

// x/v: the prev is not solely decodable, the next is solely decodable, prev+next is not decodable
// vv: the prev/next is incorporative decodable
// v/v: the prev/next is solely decodable
// xx: the prev/next is neither solely decodable nor incorporative decodable

int main(){
    // string s = "226";           // output 3   2|2|6, 22|6, 2,26
    // string s = "206";           // output 1   20|6
    // string s = "227";           // output 2   2|2|7, 22|7
    // string s = "2024";          // output 2   20|2|4, 20|24
    string s = "2124";             // output 5   2|1|2|4, 21|2|4, 21|24, 2|12|4, 2|1|24
    cout << solve(s) << endl;
    return 0;
}
```
{% endraw %}
