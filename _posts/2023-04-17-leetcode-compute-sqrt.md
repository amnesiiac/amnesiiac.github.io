---
layout: post
title: "compute sqrt (leetcode 101 binary search)"
author: "twistfatezz"
date: 2023-04-17 21:51
categories: "2023"
tags:
  - leetcode
---

### # binary search algorithm (variant of double pointer algorithm)
compute sqrt(num) for a given number <=> x^2 = num <=> compute the zero of f(x) = x^2 - num <br>
f(0)<0, f(num)>0, monotonically increaing => 0<zero<a.

{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int compute(int &num){
    if(num==0){return 0;}
    else{
        int l=0; int r=num; int mid=0; int sqrt = 0;
        while(l<=r){
            mid = l+(r-l)/2; // mid=(l+r)/2 may lead to overflow problem
            sqrt = num/mid;
            if(sqrt == mid){
                return sqrt; // #1 get sqrt when sqrt==mid
            }
            if(mid > sqrt){ // mid should be smaller, move right pointer
                r = mid-1;
            }
            if(mid < sqrt){ // mid should be bigger, move left pointer
                l = mid+1;
            }
        }
        return r; // #2 get sqrt when r<l (interleave with each other)
    }
}

int main(){
    int num1 = 8;  // testcase 1
    int num2 = 198;  // testcase 2
    std::cout << "the sqrt is: " << compute(num1) << std::endl;
    std::cout << "the sqrt is: " << compute(num2) << std::endl;
    return 0;
}
```
{% endraw %}
```txt
# output:
the sqrt is: 2
the sqrt is: 14
```

### # note
$1 how to move left/right pointer, the condition to move them. <br>
$2 the end condition to solve the problem, need to list all the situations.
