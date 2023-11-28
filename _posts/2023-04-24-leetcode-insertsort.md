---
layout: post
title: "insert sort algorithm (leetcode 101)"
author: "twistfatezz"
date: 2023-04-24 22:30
categories: "2023"
tags:
  - leetcode
---
### # insertsort algorithm
1: assume the left of element vec[i] is a sorted array. <br>
2: for element vec[j=i], if vec[j]<vec[j-1], then swap them to get them to the right order. <br>
3: looping step 2 till j=1. <br>
4: looping step 1 to step 3 till i=vec.size()-1. <br>

<hr>

### # performance analysis 
• Stable: preserve the relative order of equal elements compare to the init array. <br>
• The time complexity: O(n2), the time complexity for choosing i step is O(n), the time complexity to insert to right place is O(n). <br>
• The average space complexity: O(1), the insert algotithm only swap in place.

<hr>

### # insertsort algorithm (stable, average performance: )
{% raw %}
```cpp
#include <vector>
#include <iostream>
using namespace std;

void insertsort(vector<int> &vec, int n){
    for(int i=0; i<n; ++i){ // assume left of i already sorted
        for(int j=i; j>0 && vec[j]<vec[j-1]; j--){ // check from i to 1
            // swap
            int tmp = vec[j-1];
            vec[j-1] = vec[j];
            vec[j] = tmp;
        }
    }
}

int main(){
    vector<int> vec = {6, 3, 5, 1, 2, 10, 5, 7};
    insertsort(vec, 8);
    for_each(vec.begin(), vec.end(), [](int i){std::cout << i << " ";});
    return 0;
}
```

{% endraw %}
