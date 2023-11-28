---
layout: post
title: "two sum II (leetcode 101 double ptr)"
author: "twistfatezz"
date: 2023-04-14 19:50
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

vector<int> twosum(vector<int> &vec, int sum){
    if(vec.size()<=1){return {};}
    else{
        sort(vec.begin(), vec.end());
        int i=0; int j=vec.size()-1;
        while(i<j){
            int val = vec[i]+vec[j];
            if(val > sum){
                j--;
            }
            if(val < sum){
                i++;
            }
            if(val == sum){
                std::cout << i << " " << j << std::endl;
                return {i, j};
            }
        }
        return {};
    }
}

int main(){
    vector<int> vec = {2, 7, 11, 15};
    int sum = 7;        // test case 1
    int sum = 13;       // test case 2
    int sum = 100;      // test case 3
    twosum(vec, sum);
    return 0;
}
```
{% endraw %}
```txt
# output:
0 2
```
