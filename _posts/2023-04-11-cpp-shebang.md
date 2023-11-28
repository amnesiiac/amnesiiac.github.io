---
layout: post
title: "compile & build & run cpp like shell script (c++)"
author: "twistfatezz"
date: 2023-04-11 21:05
categories: "2023"
tags:
  - cpp
---

### # steps
```text
$ vim test.cpp
$ chmod +x test.cpp
$ ./test.cpp         # can automatically run/build
```

<hr>

### # sample file
```cpp
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <iostream>
using namespace std;

int main(){
    std::cout << "hello world" << std::endl;
    return 0;
}
```
