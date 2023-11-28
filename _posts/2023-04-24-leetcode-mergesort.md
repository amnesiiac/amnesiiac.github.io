---
layout: post
title: "merge sort algorithm (leetcode 101)"
author: "twistfatezz"
date: 2023-04-24 22:29
categories: "2023"
tags:
  - leetcode
---

### # mergesort algorithm
1: divide the pending array by the middle: l+(r-l)/2 into 2 sub arrays. <br>
2: recursively merge sort the 2 sub array. <br>
3: merge the 2 sorted sub array into a temporary array by two pointer method. <br>
4: assign the temp array to the original array.

<hr>

### # performance analysis 
• Stable: preserve the relative order of equal elements compare to the init array. <br>
• The time complexity: O(nlog2n), the time complexity for the merge step is O(n), and recursive binary division need O(log2n). <br>
• The average space complexity: O(n), it need additional space for the temporary array to hold the middle results.

<hr>

### # mergesort algorithm (stable, average performance: )
{% raw %}
method-1: divide && conquer, recursive.
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <vector>
#include <iostream>
using namespace std;

void mergesort(vector<int> &vec, int l, int r, vector<int> &tmp) {
    if(l+1 >= r){return;}
    // divide
    int m = l + (r-l)/2; // m = l + (r-l)>>1
    mergesort(vec, l, m, tmp);
    mergesort(vec, m, r, tmp);
    // conquer
    int p=l; int q=m; int i=l;
    while(p<m || q<r){ // generate tmp from [l,r)
        if(q>=r || (p<m && vec[p]<=vec[q])){ // if right part exhausted, or left<right
            tmp[i++] = vec[p++];
        }
        else{
            tmp[i++] = vec[q++];
        }
    }
    for(int j=l; j<r; j++){ // assign tmp to vec
        vec[j] = tmp[j];
    }
}
int main(){
    vector<int> vec = {6, 3, 5, 1, 2, 10, 5, 7};
    vector<int> tmp(vec.size(), 0);
    mergesort(vec, 0, 8, tmp);
    for_each(vec.begin(), vec.end(), [](int i){std::cout << i << " ";});
    return 0;
}
```
method-2: divide && conquer, iterative.
ongoing...

{% endraw %}
