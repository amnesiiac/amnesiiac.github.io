---
layout: post
title: "01/complete pack problem (leetcode 101 dynamic programming)"
author: "melon"
date: 2023-09-07 07:17
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
leetcode 101 - 01 pack problem / complete pack problem  

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

// 01 pack problem
// dp[i][j]: for total first i objs, with max weight as j, the max value sumup
int solve1(vector<int>& weight, vector<int>& value, int num, int capcity){
    vector<vector<int>> dp(num+1, vector<int>(capcity+1, 0));
    int w;
    int v;
    for(int i=1; i<=num; i++){                                // for each obj
        w = weight[i-1]; v = value[i-1];
        for(int j=1; j<=capcity; j++){                        // for each pack capcity
            if(j>=w){                                         // the i-th obj can be packed
                dp[i][j] = max(dp[i-1][j], dp[i-1][j-w]+v);   // no pack the i-th obj
            }                                                 // pack the i-th obj
            else{                                             // the i-th obj cannot be packed
                dp[i][j] = dp[i-1][j];                        // not possible to pack i-th obj
            }
            print2dvec(dp);                                   // debug
            cout << endl;
        }
    }
    return dp[num][capcity];
}

// complete pack problem
// dp[i][j]: for total first i objs, with max weight as j, the max value sumup
int solve2(vector<int>& weight, vector<int>& value, int num, int capcity){
    vector<vector<int>> dp(num+1, vector<int>(capcity+1, 0));
    int w;
    int v;
    for(int i=1; i<=num; i++){                             // for each obj
        w = weight[i-1]; v = value[i-1];
        for(int j=1; j<=capcity; j++){                     // for each pack capcity
            if(j>=w){                                      // the i-th obj can be packed
                dp[i][j] = max(dp[i-1][j], dp[i][j-w]+v);  // max(no pack i, pack i): i could be dup
            }
            else{                                          // the i-th obj cannot be packed
                dp[i][j] = dp[i-1][j];                     // the same as above 
            }
            print2dvec(dp);
            cout << endl;
        }
    }
    return dp[num][capcity];
}

// dp state transition: see solve1/solve2 inline
int main(){
    vector<int> weight = {1, 2, 3, 4, 5};                 // weight of each obj
    vector<int> value = {5, 4, 3, 2, 1};                  // value of each obj
    int num = 5;                                          // num of obj
    int capcity = 4;                                      // capcity of pack
    cout << solve1(weight, value, num, capcity) << endl;
    cout << solve2(weight, value, num, capcity) << endl;
    return 0;
}
```
```txt
2
```
{% endraw %}

<hr>

### # space compression code (dp[i][j] -> dp[j])
compress the above 01/complete pack problem 2d to 1d.
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

// 01 pack problem: each ele in line i is only rely on line i-1
//   j 0 1 2 3 ...
// i 0 * . * .
//   1 . . @ .    // the @ is depend on the * only in above line
//   2 . . . .
//     <=====  update will not overwrite "to-be used" value
int solve1(vector<int>& weight, vector<int>& value, int num, int capcity){
    vector<int> dp(capcity+1, 0);               // only hold i-1 for computing dp val of i line
    int w = -1;
    int v = -1;
    for(int i=1; i<=num; i++){
        w = weight[i-1]; v = value[i-1];
        for(int j=capcity; j>=1; j--){          // mind the traversal order -> can be simplified as j>=w
            if(j>=w){
                dp[j] = max(dp[j], dp[j-w]+v);
            }
            else{
                dp[j] = dp[j];
            }
        }
    }
    return dp[capcity];
}

// complete pack problem: each ele in line i is rely on line i/i-1
//   j 0 1 2 3 ...
// i 0 . . * .
//   1 * . @ .    // the @ is depend on the * in i/i-1 line
//   2 . . . .
//     =====>  update will not overwrite "to-be used" value
int solve2(vector<int>& weight, vector<int>& value, int num, int capcity){
    vector<int> dp(capcity+1, 0);               // only hold i-1 for computing dp val of i line
    int w = -1;
    int v = -1;
    for(int i=1; i<=num; i++){
        w = weight[i-1]; v = value[i-1];
        for(int j=1; j<=capcity; j++){          // mind the traversal order, can be simplified as j=w
            if(j>=w){
                dp[j] = max(dp[j], dp[j-w]+v);
            }
            else{
                dp[j] = dp[j];
            }
        }
    }
    return dp[capcity];
}


// 01 pack problem
// dp[i][j]: given (0,i-1) obj, the wholesome pack capcity sumup as j, then max value the pack can hold
// state transition: dp[i][j] = max(dp[i-1][j]+value[i], dp[i-1][j-weight[i]]) => whether take obj i or not
int main(){
    vector<int> weight = {1, 2, 3, 4, 5};                 // weight of each obj
    vector<int> value = {5, 4, 3, 2, 1};                  // value of each obj
    int num = 5;                                          // num of obj
    int capcity = 5;                                      // capcity of pack
    cout << solve1(weight, value, num, capcity) << endl;  // 9
    cout << solve2(weight, value, num, capcity) << endl;  // 25
    return 0;
}
```
