---
layout: post
title: "assign cookies (leetcode 101 greedy)"
author: "twistfatezz"
date: 2023-04-13 22:16
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

unsigned int total_number_candies(vector<int> &vec){
    if(vec.size()==0){return 0;}
    if(vec.size()==1){return 1;}
    vector<int> arrange(vec.size(), 1); // init
    int prev = 0;
    for(int i=1; i<vec.size(); i++){// check from left->right
        if(vec[i]>vec[prev] && arrange[i]<=arrange[prev]){// more score but <= candies
            arrange[i] = arrange[prev]+1;
        }
        prev++;
    }
    std::cout << "from left->right the arrange arr: ";
    for_each(arrange.begin(), arrange.end(), [](int i){std::cout << i << " ";});
    prev = vec.size()-1;
    for(int i=vec.size()-2; i>=0; i--){// check from right->left
        if(vec[i]>vec[prev] && arrange[i]<=arrange[prev]){// more score but <= candies
            arrange[i] = arrange[prev]+1;
        }
        prev--;
    }
    std::cout << std::endl << "from right->left the arrange arr: ";
    for_each(arrange.begin(), arrange.end(), [](int i){std::cout << i << " ";});
    return std::accumulate(arrange.begin(), arrange.end(), 0); // #include <numeric>
}

int main(){
    // vector<int> score = {1, 0, 2};// test case 1
    vector<int> score = {1, 0, 2, 1};// test case 2
    int num = total_number_candies(score);
    std::cout << std::endl << "total number of candies needed: " << num << std::endl;
    return 0;
}
```
{% endraw %}
```txt
# output:
from left->right the arrange arr: 1 1 2 1 
from right->left the arrange arr: 2 1 2 1 
total number of candies needed: 6
```
