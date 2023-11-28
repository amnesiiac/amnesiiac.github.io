---
layout: post
title: "non-overlapping intervals (leetcode 101 greedy)"
author: "twistfatezz"
date: 2023-04-13 20:14
categories: "2023"
tags:
  - leetcode
---

### # greedy algorithm 
{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

typedef vector<vector<int>> dvec;

void print(const dvec &v){ // prinf func for 2d vector
    for_each(v.begin(), v.end(), [](vector<int> tmp){
        for_each(tmp.begin(), tmp.end(), [](int i){std::cout << i <<" ";}); std::cout << "| ";
    });
    std::cout << std::endl;
}

unsigned int preserve_most_non_overlapping(dvec &v){
    if(v.size()<=1){ // escape case
        return 0;
    }
    sort(v.begin(), v.end(), [](vector<int> v1, vector<int> v2){return v1[1]<v2[1];});
    unsigned int count = 0;
    int prev = v[0][1];
    for(int i=1; i<v.size(); i++){
        if(v[i][0]<prev){
            count++;
            std::cout << " " << v[i][0] << " " << v[i][1] << ",";
        }
        else{
            prev = v[i][1];
        }
    }
    return count;
}

int main(){
    // dvec vec = {};                    // test case 1
    // dvec vec = {{1,3}, {2,4}, {1,2}}; // test case 2
    dvec vec = {{1,3}, {2,4}, {1,2}, {5,6}, {5,9}, {6,10}}; // test case 3
    std::cout << "deleted sub vectors:";
    unsigned int res = preserve_most_non_overlapping(vec);
    std::cout << std::endl;
    std::cout << "number of deleted vectors: " << res << std::endl;
    return 0;
}
```
{% endraw %}
```txt
# ./test.cpp success in centos 7
# output:
deleted sub vectors: 1 3, 5 9,
number of deleted vectors: 2
```

