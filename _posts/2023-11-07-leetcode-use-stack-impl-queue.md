---
layout: post
title: "queue implementation using stack (leetcode 101 stl stack & queue)"
author: "melon"
date: 2023-11-07 20:44
categories: "2023"
tags:
  - leetcode
---

### # theme description
leetcode 101 - 232 queue implementation using stack:  
implement a FIFO queue using only two stacks. the queue should support all the functions of a normal queue (push, peek, pop, and empty).  
1) void push(int x) Pushes element x to the back of the queue.  
2) int pop() Removes the element from the front of the queue and returns it.  
3) int peek() Returns the element at the front of the queue.  
4) boolean empty() Returns true if the queue is empty, false otherwise.

<hr>

### # code
{% raw %}
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <stack>
#include <iostream>
#include <algorithm>
using namespace std;

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){ cout << ele << " "; });
    cout << endl;
}

template <typename T>
void print2dvec(const vector<vector<T>>& vec){
    for(auto subvec : vec){
        for_each(subvec.begin(), subvec.end(), [](T ele){ cout << ele << " "; });
        cout << endl;
    }
    cout << endl;
}

class myqueue{
private:
    stack<int> in, out;
public:
    myqueue(){}
    void push(int val){
        in.push(val);
    }
    int peek(){
        in2out();
        return out.top();
    }
    void pop(){
        in2out();
        out.pop();
    }
    bool empty(){
        return in.empty() && out.empty();
    }
    void in2out(){
        while(!in.empty()){
            int tmp = in.top();
            in.pop();
            out.push(tmp);
        }
    }
};

int main(){
    myqueue que;
    que.push(1);
    que.push(2);
    que.push(3);
    cout << que.peek() << endl;  // 1
    que.pop();
    cout << que.peek() << endl;  // 2
    que.pop();
    que.pop();
    cout << que.empty() << endl; // 1
    return 0;
}
```
{% endraw %}
