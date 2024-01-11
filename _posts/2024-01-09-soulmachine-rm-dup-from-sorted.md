---
layout: post
title: "remove duplicates from sorted array (soulmachine)"
author: "melon"
date: 2024-01-09 22:42
categories: "2024"
tags:
  - leetcode
---

### # description
2.1.1 Given a sorted array, remove the duplicates in place such that each element appear only once and return the new length.
Do not allocate extra space for another array, you must do this in place with constant memory.  
For example, Given input array A = [1,1,2], should return length = 2, and A is now [1,2].

<hr>

### # solution
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

template <typename T>
int solve(vector<T>& vec){
    if(vec.size()<=1){return vec.size();}
    int curidx = 0;                       # init tail idx
    for(int i=1; i<vec.size(); i++){
        if(vec[i] != vec[curidx]){
            vec[++curidx] = vec[i];       # move ele in right pos, reset tail ele idx
        }
    }
    return curidx+1;                      # return len of vec
}

// remove duplicate elements from sorted array
// A = [1, 1, 2];
int main(){
    // vector<int> vec = {1, 1, 2, 3, 3, 4};
    // vector<int> vec = {1, 1, 2, 3, 3, 4, 4, 4, 5};
    vector<int> vec = {1, 1};
    int res = solve(vec);
    cout << res << endl;
    return 0;
}
```
