---
layout: post
title: "bubble sort algorithm (leetcode 101)"
author: "twistfatezz"
date: 2023-04-24 22:37
categories: "2023"
tags:
  - leetcode
---

### # bubblesort algorithm
1: there are vec.size()-1 elements in the array, so there vec.size()-2 times to perform bubble floating, for each time, looping from step 2 to step 4: <br>
2: start at the first adjacent pair (0,1). <br>
3: compare & swap if the vec[0]<vec[1]. <br>
4: move to the next adjacent pair. <br>
5: each looping will make the last item the biggest, thus we could loop only the unsorted elements. <br>

<hr>

### # performance analysis 
• Stable: preserve the relative order of equal elements compare to the init array. <br>
• The time complexity: O(n^2), for each compare in for loop, a swap is needed, bubblesort is inefficient but easy to understand. <br>
• The average space complexity: O(1), the algorithm sort array in place (constant space), and there are no space need for recursive stack.

<hr>

### # bubblesort algorithm implementation
{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

void bubblesort(vector<int> &vec){
    bool swap; int tmp;
    for(std::size_t i=1; i<vec.size(); i++){ // i: the times of bubble swap to the 'end'
        swap = false;
        for(std::size_t j=1; j<vec.size()-i+1; j++){ // each time the last i element are top-k largest
            if(vec[j]<vec[j-1]){ // swap for reverse order elements
                tmp = vec[j-1]; vec[j-1]=vec[j]; vec[j]=tmp;
                swap = true;
            }
        }
        if(!swap){ // if no swap, the array is sorted
            return;
        }
    }
}

int main(){
    vector<int> vec = {2, 5, 6, 0, 0, 1, 2};
    std::cout << "the original vector: ";
    for_each(vec.begin(), vec.end(), [](int i){std::cout << i << " ";});
    std::cout << std::endl;
    bubblesort(vec);
    std::cout << "the vector sorted: ";
    for_each(vec.begin(), vec.end(), [](int i){std::cout << i << " ";});
    std::cout << std::endl;
    return 0;
}
```
{% endraw %}
