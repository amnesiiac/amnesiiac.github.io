---
layout: post
title: "selection sort algorithm (leetcode 101)"
author: "twistfatezz"
date: 2023-04-24 23:04
categories: "2023"
tags:
  - leetcode
---

### # selectionsort algorithm
1: assume the whole array is a unsorted array. <br>
2: find the min of the unsorted array, then swap it with the first element in the array. <br>
3: find the min of the left unsorted array, repeat this to the end. <br>

<hr>

### # performance analysis 
• Unstable: does not preserve the relative order of equal elements compare to the init array. <br>
• The time complexity: O(n2), the time complexity for choosing base element is O(n), the time complexity to find min & swap is O(n). <br>
• The average space complexity: O(1), the insert algotithm only swap in place.

<hr>

### # selectionsort algorithm (unstable, average performance: )
{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

void selectionsort(vector<int> &vec){
    std::size_t base; int tmp;
    for(std::size_t i=0; i<vec.size(); i++){ // for each base ele (the eles before i is sorted)
        base = i;
        for(std::size_t j=i+1; j<vec.size(); j++){ // find the smallest behind i
            if(vec[j] < vec[base]){
                base = j;
            }
        }
        tmp=vec[base]; vec[base]=vec[i]; vec[i]=tmp; // swap
    }
}

int main(){
    vector<int> vec = {2, 5, 6, 0, 0, 1, 2};
    std::cout << "the original vector: ";
    for_each(vec.begin(), vec.end(), [](int i){std::cout << i << " ";});
    std::cout << std::endl;
    selectionsort(vec);
    std::cout << "the vector sorted: ";
    for_each(vec.begin(), vec.end(), [](int i){std::cout << i << " ";});
    std::cout << std::endl;
    return 0;
}
```

{% endraw %}
