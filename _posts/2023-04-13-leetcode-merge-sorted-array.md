---
layout: post
title: "merge sorted array (leetcode 101 double ptr)"
author: "twistfatezz"
date: 2023-04-13 22:02
categories: "2023"
tags:
  - leetcode
---

### # double pointer algorithm
{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

void merge(vector<int> &v1, vector<int> &v2, int m, int n){// available elements in vec1/2
    int i=v1.size()-1; // currently filling element
    while(m>=0 && n>=0){
        if(v1[m-1] < v2[n-1]){
            v1[i--] = v2[n-1];
            n--;
        }
        else{
            v1[i--] = v1[m-1];
            m--;
        }
    }
}

int main(){
    //vector<int> v1 = {1,2,2,3,4,5,0,0,0};
    //vector<int> v2 = {0,9,10};
    vector<int> v1 = {1,2,2,0,0,0,0};
    vector<int> v2 = {0,9,10,11};
    int m = 3; int n = 4; // number of elements
    merge(v1, v2, m, n);
    for_each(v1.begin(), v1.end(), [](int i){std::cout << i << " ";});
    std::cout << std::endl;
    return 0;
}
```
{% endraw %}
```txt
# output:
0 1 2 2 9 10 11
```
