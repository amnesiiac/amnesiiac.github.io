---
layout: post
title: "different ways to add parentheses (leetcode 101 dp/divide_conquer)"
author: "melon"
date: 2023-10-25 22:08
categories: "2023"
tags:
  - leetcode
---

### # divide and conquer

<hr>

### # dynamic programming algorithm
(1) construct "state transition metric".  
(2) make sure the boundary condition that could startup the "transition".  
(3) the state transition metric value stores the userful result to solve the problem. Analysis the state transition graphs, firstly determine the type of operations can meet the requirement, then find out the "start state" matches each conversion operation, finally the wholesome state transition metric is derived.  
(4) the startup condition is the boundary condition.

<hr>

### # theme description
leetcode 101 - 241 different ways to add parentheses 

<hr>

### # code
recursive divide and conquer algorithm:
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
// #include <bits/stdc++.h>
#include <vector>
#include <unordered_set>
#include <string>
#include <iostream>
using namespace std;

template <typename T>
void print1dvec(const vector<T>& vec){
    for_each(vec.begin(), vec.end(), [](T ele){ cout << ele << " "; });
    cout << endl;
}

void solve(unordered_set<int>& res, const string& str){  // return vec of possible val computed
    char c;
    int l, r;
    for(int i=0; i<str.length(); i++){        // for each startup char
        c = str[i];
        if(c=='*' || c=='+' || c=='-'){       // for each possible operators
            unordered_set<int> left, right;
            solve(left, str.substr(0, i));    // derive left operand all possible val
            solve(right, str.substr(i+1));    // derive right operand all possible val
            for(int& l: left){
                for(int& r: right){
                    switch(c){
                        case '*':
                            res.insert(l*r);
                            break;
                        case '+':
                            res.insert(l+r);
                            break;
                        case '-':
                            res.insert(l-r);
                            break;
                    }
                }
            }
        }
    }
    if(res.empty()){                          // edge condition: only a single char as input str
        res.insert(stoi(str));                // directly add it into 
    }
}

int main(){
    string input = "2-1-1*2-9";
    unordered_set<int> res;                   // use unordered_set to keep unique val of computes result
    solve(res, input);
    for(auto item: res){                      // 0: (2-1)-1;    2: 2-(1-1)
        cout << item << " ";
    }
    return 0;
}
```
todo: dynamic programming methods, draw state transition graph, then programming:
