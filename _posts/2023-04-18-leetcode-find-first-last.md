---
layout: post
title: "find the first/last ele to meet the condition (leetcode 101 binarysearch)"
author: "twistfatezz"
date: 2023-04-18 20:39
categories: "2023"
tags:
  - leetcode
---

### # binary search algorithm (variant of double pointer algorithm)

{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

// find the lower bound => '['
int lower_bound(const vector<int> &vec, int &target){
    int l=0; int r=vec.size(); int mid=0;     // find range, so init right pointer as vec.size()
    while(l<r){                               // if l=r, reach the return condition
        mid = l+(r-l)/2;
        if(vec[mid] >= target){               // mind the =
            r = mid;
        }
        if(vec[mid] < target){
            l = mid+1;
        }
    }
    return l;
}

// find the upper bound => ')'
// don not try to find the ']', will fall in infinite loop if value not exist in arr
int upper_bound(const vector<int> &vec, int &target){
    int l=0; int r=vec.size(); int mid;       // find a range, so init right pointer as vec.size()
    while(l<r){                               // if l=r, reach the return condition
        mid = l+(r-l)/2; 
        if(vec[mid] > target){
            r = mid;
        }
        if(vec[mid] <= target){               // mind the =
            l = mid+1;
        }
    }
    return r;
}

int main(){
    vector<int> vec = {5, 7, 7, 8, 8, 10};    // testcase 1
    // vector<int> vec = {5, 5, 6, 6, 7, 7};  // testcase 2
    int target = 8;
    int low = lower_bound(vec, target);
    int up = upper_bound(vec, target)-1;
    if(low>=vec.size() || vec[low]!=target){ // add judge for 404 not found 
        std::cout << "the low/up range: " << -1 << " " << -1 << std::endl;
    }
    else{
        std::cout << "the low/up range: " << low << " " << up << std::endl;
    }
    return 0;
}
```
output:
```text
the low/up range: 3 4
the low/up range: -1 -1
```
{% endraw %}

