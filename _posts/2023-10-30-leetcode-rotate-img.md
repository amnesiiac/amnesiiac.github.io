---
layout: post
title: "rotate img (leetcode 101 stl vector)"
author: "melon"
date: 2023-10-30 21:15
categories: "2023"
tags:
  - leetcode
---

### # theme description
leetcode 101 - 48 rotate image:  
You are given an nxn 2D matrix representing an image, rotate the image by 90 degrees (clockwise).  
You have to rotate the image in-place, which means you have to modify the input 2D matrix directly.
Do not allocate another 2D matrix and do the rotation.

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <iostream>
using namespace std;

template <typename T>
void print2dvec(const vector<vector<T>>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T ele){ cout << ele << " "; });
        cout << endl;
    }
    cout << endl;
}


// each time we pay attention to only the ele of four related pos
//   *            *:        (0,0) -> (0,2)   -> (2,2)     -> (2,0)
//   1  2  3 *    formular: (i,j) -> (j,n-i) -> (n-i,n-j) -> (n-j,i)
//   4  5  6
// * 7  8  9
//         *
// when coding, mind the i/j ranges
void solve(vector<vector<int>>& vec){
    int n = vec.size()-1;
    int tmp;
    for(int i=0; i<=n/2; i++){            // column of an ele to get handled
        for(int j=i; j<n-i; j++){         // row of the ele to get handled
            tmp = vec[i][j];              // for inplace modification
            vec[i][j] = vec[n-j][i];
            vec[n-j][i] = vec[n-i][n-j];
            vec[n-i][n-j] = vec[j][n-i];
            vec[j][n-i] = tmp;
        }
    }
}

// rotate img by 90 degree clockwise, the changes should be made in place.
int main(){
    // vector<vector<int>> img = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
    vector<vector<int>> img = {{1, 2, 3, 4}, {5, 6, 7, 8}, {9, 10, 11, 12}, {13, 14, 15, 16}};
    solve(img);
    print2dvec(img);
    return 0;
}
```
```text
7 4 1 
8 5 2 
9 6 3 

13 9 5 1 
14 10 6 2 
15 11 7 3 
16 12 8 4 
```
{% endraw %}
