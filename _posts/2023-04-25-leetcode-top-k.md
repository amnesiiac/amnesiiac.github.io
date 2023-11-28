---
layout: post
title: "top k element (leetcode 101 bubble sort)"
author: "twistfatezz"
date: 2023-04-25 21:18
categories: "2023"
tags:
  - leetcode
---

### # bubble sort to get top-k algorithm
{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int topk(vector<int> &vec, int k){
    int tmp;
    for(std::size_t i=1; i<vec.size(); i++){
        for(std::size_t j=1; j<vec.size()-i+1; j++){
            if(vec[j]<vec[j-1]){
                tmp=vec[j-1]; vec[j-1]=vec[j]; vec[j]=tmp;
            }
        }
        if(int(i)==k){return vec[vec.size()-k];}
    }
    return -1;
}

int main(){
    vector<int> vec = {2, 5, 6, 0, 0, 1, 2, 9, 10, 11, 1000, -1, -2, -3, 1001};
    std::cout << "the topk: " << topk(vec, 2) << std::endl;
    std::cout << "topk aligned rightwards: ";
    for_each(vec.begin(), vec.end(), [](int i){std::cout << i << " ";}); // 2011205
    std::cout << std::endl;
    return 0;
}
```
```txt
the topk: 1000
topk aligned rightwards: 2 0 0 1 2 5 6 9 10 -1 -2 -3 11 1000 1001
```

<hr>

### # quick selection + binary search to get topk (performant solution)
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int candidate(vector<int> &vec, int l, int r){ // quick selection method
    int first=l; int last=r-1; int key=vec[first];
    while(first < last){
        while(first<last && vec[last]>=key){
            last--;
        }
        vec[first] = vec[last];
        while(first<last && vec[first]<=key){
            first++;
        }
        vec[last] = vec[first];
    }
    vec[first] = key;
    return last;
}


int topk(vector<int> &vec, int k){ // binary search to get the topk
    int l=0; int r=vec.size(); int mid; int target = vec.size()-k; // topk index
    while(l<r){
        mid = candidate(vec, l, r);
        if(mid == target){
            return vec[mid];
        }
        if(mid < target){
            l = mid+1;
        }
        if(mid > target){
            r = mid-1;
        }
    }
    return vec[l];
}

int main(){
    vector<int> vec = {2, 5, 6, 0, 0, 1, 2, 9, 10, 11, 1000, -1, -2, -3, 1001};
    std::cout << "the topk: " << topk(vec, 3) << std::endl;
    return 0;
}
```
```txt
the topk: 11
```
{% endraw %}
