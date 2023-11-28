---
layout: post
title: "combinations (leetcode 101 backtracking)"
author: "melon"
date: 2023-07-17 22:26
categories: "2023"
tags:
  - leetcode
---

### # backtracking algorithm

<hr>

### # theme description

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

// output combination of 2 ele out of 1 2 3 4
void combination(vector<vector<int>>& res, vector<int>& tmp, int& count, int start, int n, int k){
    if(count==k){ // terminate condition
        res.push_back(tmp);
    }
    else{
        for(int i=start; i<=n; i++){                  // the range is from 1 to 4
            tmp[count++] = i;                         // one step forward
            combination(res, tmp, count, i+1, n, k);  // recursive to finish the rest job
            count--;                                  // return back
        }
    }
}

int main(){
    int n=4; int k=2; int count=0; int start=1;
    vector<vector<int>> res;
    vector<int> tmp(k, 0);
    combination(res, tmp, count, start, n, k);
    for(auto vec: res){
        for_each(vec.begin(), vec.end(), [](int i){std::cout << i << " ";});
        cout << endl;
    }
    return 0;
}
```
output:
```txt
1 2
1 3
1 4
2 3
2 4
3 4
```

{% endraw %}
