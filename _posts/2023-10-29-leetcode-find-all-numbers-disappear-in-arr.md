---
layout: post
title: "find all number disappeared in an arr (leetcode 101 stl vector)"
author: "melon"
date: 2023-10-29 18:36
categories: "2023"
tags:
  - leetcode
---

### # theme description
leetcode 101 - 448 find all disappeared number in an arr:  
given an array nums of n integers where nums[i] is in the range [1, n], return an array of all the integers in the range [1, n] that do not appear in nums.

<hr>

### # code
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <iostream>
using namespace std;

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){ cout << ele << " "; });
    cout << endl;
}

// marking rules:
// if the element is appeared, set its corresponding ele: vec[idx] < 0
// else, keep it as original >0
void solve(vector<int>& vec, vector<int>& ans){
    int pos;
    for(int i=0; i<vec.size(); i++){
        pos = abs(vec[i])-1;          // get the pos to set, abs to correct negative val back
        if(vec[pos]>0){               // only mark positive element
            vec[pos] = -vec[pos];     // mark the pos corresponding ele as its opposite
        }
    }
    for(int j=0; j<vec.size(); j++){  // get the marked idx out
        if(vec[j]>0){
            ans.push_back(j+1);
        }
    }
}

// 448 find all numbers disappeared in a array, output arr contains nums not appear in vec
// idx:         0  1  2  3  4  5  6  7
// vec:         4  3  2  7  8  2  3  1
// after mark: -4 -3 -2 -7  8  2 -3 -1 
int main(){
    vector<int> vec = {4, 3, 2, 7, 8, 2, 3, 1};
    vector<int> ans;
    solve(vec, ans);
    print1dvec(ans);
    return 0;
}
```
