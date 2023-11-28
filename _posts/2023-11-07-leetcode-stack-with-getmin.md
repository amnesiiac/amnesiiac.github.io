---
layout: post
title: "stack implementation with getmin (leetcode 101 stl stack & queue)"
author: "melon"
date: 2023-11-07 19:40
categories: "2023"
tags:
  - leetcode
---

### # theme description
leetcode 101 - 155 implemetation stack with getmin method:  
design a stack that supports push, pop, top, and retrieving the minimum element in constant time.  
1) minstack() initializes the stack object.  
2) void push(int val) pushes the element val onto the stack.  
3) void pop() removes the element on the top of the stack.  
4) int top() gets the top element of the stack.  
5) int getmin() retrieves the minimum element in the stack.  
implement each function with O(1) time complexity.

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){ cout << ele << " "; });
    cout << endl;
}

class minstack{
private:
    stack<int> stk, minstk;                       // maintain both stack for ele & minval
public:
    minstack(){}
    void push(int val){
        stk.push(val);                            // update ele stack
        if(!minstk.empty() && minstk.top()<val){  // update minstk
            minstk.push(minstk.top());
        }
        else{
            minstk.push(val);
        }
    }
    void pop(){                                   // update ele & minstk
        stk.pop();
        minstk.pop();
    }
    int top(){
        return stk.top();
    }
    int getmin(){
        return minstk.top();
    }
};

int main(){
    minstack mystk;
    mystk.push(-2);
    mystk.push(0);
    mystk.push(-3);
    cout << "getmin: " << mystk.getmin() << endl;
    mystk.pop();
    mystk.push(-5);
    cout << "top: " << mystk.top() << endl;
    cout << "getmin: " << mystk.getmin() << endl;
    mystk.pop();
    mystk.pop();
    mystk.pop();
    cout << "top: " << mystk.top() << endl;        // return segmentation fault, could add check before return
    cout << "getmin: " << mystk.getmin() << endl;
    return 0;
}
```
```text
getmin: -3
top: -5
getmin: -5
Segmentation fault
```
{% endraw %}
