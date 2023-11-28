---
layout: post
title: "top k frequent elements (leetcode 101 bucket sort)"
author: "twistfatezz"
date: 2023-05-06 21:45
categories: "2023"
tags:
  - leetcode
---

### # using buckets sort to get top frequent elements
{% raw %}
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <vector>
#include <unordered_map>
#include <iostream>
using namespace std;

void print(const vector<vector<int>> &vec){
    for(const auto &v : vec){
        for(const auto &e : v){
            std::cout << e << " ";
        }
        std::cout << std::endl;
    }
}

vector<int> topk(vector<int> &vec, int k){
    int max_count = 0;
    unordered_map<int, int> counts; // store the ele->count mapping
    for(const int &i:vec){
        max_count = max(max_count, ++counts[i]);
    }
    vector<vector<int>> buckets(max_count+1); // count->[ele1, ele2...] collect ele with same frequency
    for(const auto &p:counts){
        buckets[p.second].push_back(p.first);
    }
    std::cout << "the buckets created is: ";
    print(buckets);
    vector<int> ans;
    for(std::size_t i=max_count; i>=0 && ans.size()<k; i--){ // for each frequency/count
        for(const int &j:buckets[i]){ // for each element in the frequency=i
            ans.push_back(j);
            if(ans.size()==k){
                break;
            }
        }
    }
    return ans;
}

int main(){
    vector<int> vec = {1, 1, 1, 1, 2, 2, 3, 6};
    vector<int> out = topk(vec, 4);
    std::cout << "the topk frequent elements are: ";
    for_each(out.begin(), out.end(), [](int i){std::cout << i << " ";});
    return 0;
}
```
```txt
# index=frequency
the buckets created is: 
3 6   (1)
2     (2)
      (3)
1     (4)
the topk frequent elements are: 1 2 3 6
```
{% endraw %}
