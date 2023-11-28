---
layout: post
title: "max trunks to make sorted (leetcode 101 stl vector)"
author: "melon"
date: 2023-11-01 20:37
categories: "2023"
tags:
  - leetcode
---

### # theme description
leetcode 101 - 769 max trunks to make sorted:  
You are given an integer array arr of length n that represents a permutation of the integers in the range [0, n-1].  
We split arr into some number of chunks (i.e., partitions), and individually sort each chunk. After concatenating them, the result should equal the sorted array.  
Return the largest number of chunks we can make to sort the array.

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <iostream>
#include <algorithm>                       // for max
using namespace std;

int solve(vector<int>& vec){               // output how many trunks can be generated
    int curmax = INT_MIN;
    int count = 0;                         // trunks count
    for(int i=0; i<vec.size(); i++){
        curmax = max(curmax, vec[i]);      // record current max
        if(curmax == i){                   // if current max = current idx
            count++;                       // a trunk generated
        }
    }
    return count;
}

int main(){                                // 769: max trunks to make sorted
    // vector<int> vec = {1, 0, 2, 3, 4};
    vector<int> vec = {3, 0, 2, 1, 4};
    cout << solve(vec) << endl;
    return 0;
}
```
output:
```text
test1:
idx: 0 1 2 3 4
ele: 1 0 2 3 4
     - -|-|-|-    trunks=4

test2:
idx: 0 1 2 3 4
ele: 3 0 2 1 4
     - - - -|-    trunks=2
```
{% endraw %}
