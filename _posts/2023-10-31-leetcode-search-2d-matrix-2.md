---
layout: post
title: "search a 2d matrix II (leetcode 101 stl vector)"
author: "melon"
date: 2023-10-31 19:05
categories: "2023"
tags:
  - leetcode
---

### # theme description
leetcode 101 - 240 search a 2d matrix II:  
Write an efficient algorithm that searches for a value target in an m x n integer matrix matrix. This matrix has the following properties:  
Integers in each row are sorted in ascending from left to right.  
Integers in each column are sorted in ascending from top to bottom.

<hr>

### # code
{% raw %}
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
    cout << endl;
}

// 240: find if the target val inside 2d vec
bool solve(vector<vector<int>>& vec, int target, vector<vector<char>>& route){
    int startuprow = 0;
    for(int j=vec[0].size()-1; j>=0; j--){         // col
        for(int i=startuprow; i<vec.size(); i++){  // row
            route[i][j] = '*';                     // update search route
            if(vec[i][j] < target){
                startuprow = i+1;                  // set next col startup search row
            }
            if(vec[0][j] > target){                // goto next col
                break;
            }
            if(vec[i][j] == target){
                return true;
            }
        }
    }
    return false;
}

// 241: different way to add parenthesis
int main(){
    vector<vector<int>> vec = {
        {1, 4, 7, 11, 15}, 
        {2, 5, 8, 12, 19}, 
        {3, 6, 9, 16, 22}, 
        {10, 13, 14, 17, 24}, 
        {18, 21, 23, 26, 30}
    };
    vector<vector<char>> route(5, vector<char>(5, '-'));
    // int target = 5;
    // int target = 10;
    int target = 44;
    if(solve(vec, target, route)){
        cout << "existed" << endl;
    }
    print2dvec(route);
    return 0;
}
```
output:
```text
target = 5
existed
- * * * *
- * - - -
- - - - -
- - - - -
- - - - -

target = 10
existed
- - * * *
- - * - -
- - * - -
* * * - -
- * * - -

target = 44
- - - - *
- - - - *
- - - - *
- - - - *
- - - - *
```
{% endraw %}
