---
layout: post
title: "realpath & absolute-path (c++)"
author: "melon"
date: 2023-07-27 08:19
categories: "2023"
tags:
  - cpp
  - ongoing
---

### # code
```text
#include <string>
#include <stdlib.h>
#include <iostream>
#include <libgen.h>  // dirname

using std::string;
using std::cout;
using std::endl;

char* getrealpath(const char* symlink, size_t len=100){
    char abs_path[len];
    char actual[len+1];
    return realpath(symlink, NULL);
}

void test(){
    std::string file = __FILE__;
    std::string dir = dirname(__FILE__);
    cout << dir << endl;
}

int main(){
    // string symlink = ".";
    // /home/metung/bin/k -> /home/metung/bin/shuttle/out
    // char* symlink = "/home/metung/bin/k";
    string symlink = "/home/metung/bin/k";
    char* ret = getrealpath(symlink.c_str());
    cout << ret << endl;
    return 0;
}
```
output:
```txt
output: /home/metung/bin/shuttle/out
```
