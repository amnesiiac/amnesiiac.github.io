---
layout: post
title: "valid parentheses (leetcode 101 stl stack & queue)"
author: "melon"
date: 2023-11-08 20:49
categories: "2023"
tags:
  - leetcode
---

### # theme description
leetcode 101 - 20 valid parentheses: (to be checked)  
Given a string s containing just the characters '(', ')', '{', '}', '[' and ']', 
determine if the input string is valid. an input string is valid if:  
1) open brackets must be closed by the same type of brackets.  
2) open brackets must be closed in the correct order.  

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

bool solve(const string& str){
    stack<char> stk;
    for(int i=0; i<str.length(); i++){
        bool ispair = false;
        if(!stk.empty()){
			switch(str[i]){
                case '}':
                    ispair = stk.top() == '{'? true : false;
                    break;
                case ']':
                    ispair = stk.top() == '['? true : false;
                    break;
                case ')':
                    ispair = stk.top() == '('? true : false;
                    break;
                default:            // if str[i] in { or [ or (
                    ispair = false;
            }
            if(ispair){
                stk.pop();
            }
			else{
                stk.push(str[i]);
            }
        }
        else{
            stk.push(str[i]);
        }
    }
    return stk.empty();
}

int main(){
    string str = "{[]}()";                 // valid
    // string str = "{[]}(){{[[(())]]}}";  // valid
    // string str = "{[]}(){{[[(()]]}}";  // not valid
    if(solve(str)){
        cout << "matched";
    }
    else{
        cout << "not matched";
    }
    return 0;
}
```
{% endraw %}
