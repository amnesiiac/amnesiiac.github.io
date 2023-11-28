---
layout: post
title: "search in rotate array (leetcode 101 binary search)"
author: "twistfatezz"
date: 2023-04-19 21:04
categories: "2023"
tags:
  - leetcode
---

### # binary search algorithm (variant of double pointer algorithm)

{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

// find ele in arr
bool find_in_rotate(vector<int> &vec, int &target){ 
    int l=0; int r=vec.size()-1; int mid;
    while(l<=r){ // the following procedure can cover l=r
        mid = l+(r-l)/2;
        if(vec[mid] == target){return true;} // found the num
        if(vec[r] == vec[mid]){ // cannot judge which part to search for target
            r--; // narrow down search range
        }
        else if(vec[l] == vec[mid]){ // cannot judge too
            l++;
        }
        else if(vec[r] > vec[mid]){ // right part is monotonous
            if(target > vec[mid] && target <= vec[r]){ // target in right part
                l=mid+1;
            }
            else{r=mid-1;} // narrow down range
        }
        else{ // left part is monotonous
            if(target < vec[mid] && target >= vec[l]){ // target in left part
                l=mid+1;
            }
            else{r=mid-1;}
        }
    }
    return false;
}

int main(){
    vector<int> vec = {2, 5, 6, 0, 0, 1, 2};
    int target = 6; // testcase 1
    int target = 7; // testcase 2
    std::cout << "target existence: " << find_in_rotate(vec, target) << std::endl;
    return 0;
}
```
{% endraw %}
```txt
# output:
target existence: 1
target existence: 0
```

