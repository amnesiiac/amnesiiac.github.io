---
layout: post
title: "embed python code in cpp (c++ python)"
author: "melon"
date: 2023-06-24 21:25
categories: "2023"
tags:
  - cpp
---

### # code to run python hello world in cpp 
{% raw %}
menuitem.h: the header file to init 'json-like' key-value data.
```cpp
#include <stdio.h>
#include <iostream>
#include <Python/Python.h>  // macos include format
// #include <Python.h>      // general linux include format

int main(){
    PyObject* pInt;
    // init python interpretor
    Py_Initialize();
    // run python code in cpp
    PyRun_SimpleString("print('Hello World from Embedded Python!!!')");
    // exit python interpretor
    Py_Finalize();

    return 0;
}
```
{% endraw %}

<hr>

### # Makefile to enable execute python in cpp 
remember to use <b>-framework Python</b> to enable invoking methods in Python.h.
```makefile
CXX=g++
CFLAGS=-g -Wall -std=c++17 -pthread -w -framework Python
OUT=out

SUBDIR=$(shell ls -d */)

ROOTSRC=$(wildcard *.cpp)
ROOTOBJ=$(ROOTSRC:%.cpp=%.o)

SUBSRC=$(shell find $(SUBDIR) -name '*.cpp')
SUBOBJ=$(SUBSRC:%.cpp=%.o)

$(OUT):$(ROOTOBJ) $(SUBOBJ)
	$(CXX) $(CFLAGS) -o $@ $^
.cpp.o:
	$(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
	rm -f *.o $(OUT) $(ROOTOBJ) $(SUBOBJ)
```

<hr>

### # program ouput
```txt
Hello World from Embedded Python!!!
```

<hr>

### # reference
https://stackoverflow.com/a/16454614
