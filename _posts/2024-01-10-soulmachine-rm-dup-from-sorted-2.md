---
layout: post
title: "remove duplicates from sorted array II (soulmachine)"
author: "melon"
date: 2024-01-10 20:51
categories: "2024"
tags:
  - leetcode
---

### # description
2.1.2 Follow up for rm dup from sorted 1: what if duplicates are allowed at most twice?  
Given sorted array A = [1,1,1,2,2,3], should return length = 5, and A is now [1,1,2,2,3].

<hr>

### # solution
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <iostream>
using namespace std;

template <typename T>
int solve(vector<T>& vec){
    if(vec.size()<=1){return vec.size();}
    int curidx = 0;                        // init tail idx
    int count = 1;                         // num of idx duplicate
    for(int i=1; i<vec.size(); i++){
        if(vec[i] != vec[curidx]){
            vec[++curidx] = vec[i];        // move ele in right pos, reset tail ele idx
            count = 1;
        }
        else{
            if(++count<3){
                curidx++;
            }
        }
    }
    return curidx+1;                       // return len of vec
}

int main(){
    // vector<int> vec = {1, 1, 2, 3, 3, 4, 5, 5, 5, 5, 5};  // 8
    // vector<int> vec = {1, 1, 2, 3, 3, 4, 4, 4, 5};        // 8
    vector<int> vec = {1, 1, 1, 1, 1, 1, 1, 1};              // 2
    // vector<int> vec = {1, 1};                             // 2
    int res = solve(vec);
    cout << res << endl;
    return 0;
}
```
