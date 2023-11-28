---
layout: post
title: "deep copy & shallow copy (c++)"
author: "melon"
date: 2023-10-14 11:46
categories: "2023"
tags:
  - cpp
---

### # deep copy & shallow copy in self-maintained class
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

class Car{
public:
    string name;
    vector<string> colors;

    Car(string name, vector<string> colors){
        this->name = name;
        this->colors = colors;
    }
};

int main(){
    vector<string> honda_colors = {"Red", "Blue"};
    Car honda = Car("Honda", honda_colors);                       // init a car obj

    Car deepcopy_honda = honda;                                   // deepcopy: =
    deepcopy_honda.colors.push_back("Green");
    cout << "deepcopy: " << deepcopy_honda.colors.back() << endl; 
    cout << "original: " << honda.colors.back() << endl;
    cout << endl;

    Car* copy_honda = &honda;                                     // shallow copy: *
    copy_honda->colors.push_back("Green");
    cout << "shallowcopy: " << copy_honda->colors.back() << endl;
    cout << "orginal: " << honda.colors.back() << endl;
    return 0;
}
```
```text
$ g++ -g -Wall -Wextra -std=c++11 -O1    copy.cpp   -o copy
deepcopy: Green
original: Blue

shallowcopy: Green
orginal: Green
```
