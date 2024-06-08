---
layout: post
title: "profiling python by native module (python, profiling)"
author: "melon"
date: 2023-10-11 21:45
categories: "2023"
tags:
  - python
  - profiling
---

todo: get a more complexed multi-threading toy code to test the performance of this tool.

a sample code to profile python program memory usages:
```text
from memory_profiler import profile

@profile  # pass the func below to wrapper decorator
def func():
    x = [ x for x in range(0, 1000) ]
    y = [ y*100 for y in range(0, 1500) ]
    del x
    return y

if __name__ == '__main__':
    func()
```

the profile script output:
```text
Filename: test2.py

Line #    Mem usage    Increment  Occurrences   Line Contents
=============================================================
     4     17.5 MiB     17.5 MiB           1   @profile
     5                                         
     6                                         # code for which memory has to be monitored
     7                                         def func():
     8     17.5 MiB      0.0 MiB        1003       x = [x for x in range(0, 1000)]
     9     17.6 MiB      0.1 MiB        1503       y = [y*100 for y in range(0, 1500)]
    10     17.6 MiB      0.0 MiB           1       del x
    11     17.6 MiB      0.0 MiB           1       return y
```
