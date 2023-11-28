---
layout: post
title: "daily temperatures (leetcode 101 stl stack & queue)"
author: "melon"
date: 2023-11-09 21:00
categories: "2023"
tags:
  - leetcode
---

### # theme description
leetcode 101 - 739 daily temperatures:  
given an array of integers temperatures represents the daily temperatures, 
return an array:  
answer[i] is the number of days you have to wait after the i-th day to get a warmer temperature.  
if there is no future day for which this is possible, keep answer[i] == 0 instead.

<hr>

### # code
solution using vector:
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){cout << ele << " "; });
    cout << endl;
}

template <typename T>
void print2dvec(const vector<T>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T ele){cout << ele << " "; });
    }
    cout << endl;
}

vector<int> solve(const vector<int>& vec){
    int n = vec.size();
    if(n<2){
        return {0};
    }
    vector<int> ans;
    bool setf;
    for(int i=0; i<n; i++){          // for each day in vec
        setf = false;
        for(int j=i+1; j<n; j++){
            if(vec[j]>vec[i]){       // for each post days
                ans.push_back(j-i);
                setf = true;
                break;
            }
        }
        if(!setf){                   // cannot be determined or no warmer days
            ans.push_back(0);
        }
    }
    return ans;
}

int main(){
    vector<int> vec = {73, 74, 75, 71, 68, 72, 76, 73};
    vector<int> res = solve(vec);
    print1dvec(res);
    return 0;
}
```

solution using stack:
```text
vector<int> dailyTemperatures(vector<int>& temperatures){
    int n = temperatures.size();
    vector<int> ans(n);
    stack<int> indices;                                      // days pending for temperature comparison
    for(int i=0; i<n; ++i){                                  // for each day for handling
        while(!indices.empty()){                             // if exists days to get compared
            int pre_index = indices.top();                   // idx pending for temmperature comparison
            if(temperatures[i] <= temperatures[pre_index]){  // if current day is colder than pre idx
                break;                                       // move to next day
            }
            indices.pop();                                   // cur day is warmer, get the idx out from pending list
            ans[pre_index] = i - pre_index;                  // set the num till next warmer day of cur i
        }
        indices.push(i);                                     // add cur idx in stk prepare for comparison
    }
    return ans; 
}
```
{% endraw %}
