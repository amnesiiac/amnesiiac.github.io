---
layout: post
title: "profiling python programs using perf (python & profiling)"
author: "melon"
date: 2023-10-10 20:45
categories: "2023"
tags:
  - python
  - profiling
  - ongoing
---

### # introduction to use perf in python
The main problem with using perf with python app is that: perf only gets information about native symbols (names of functions and procedures written in c).  
So the code/file names of python functions in your code will not appear in the output of perf.

Since python 3.12, the interpreter can run in a special mode that allows python functions to appear in the output of the perf.  

When this mode is enabled, the interpreter will interpose a small piece of code compiled on the fly before the execution of every python function, 
which will teach perf the relationship between this piece of code and the associated python function using perf map files.

<hr>

### # check if perf code interposition works for your system
Note: the support for the perf is only available for linux on select architectures till 20231010.  
Check the output of following cmd to see if your system support using perf for python:
```text
$ python -m sysconfig | grep HAVE_PERF_TRAMPOLINE
HAVE_PERF_TRAMPOLINE=1
```
For best perf profiling results, python should be compiled with:
```text
CFLAGS="-fno-omit-frame-pointer -mno-omit-leaf-frame-pointer"
```
as this allows profilers to unwind using only the frame pointer but not on dwarf debug info.  
Because, the time when the code interposed to allow perf support is dynamically generated, it doesn’t have any dwarf debugging information available yet.

The method for checking whether the python is compiled with the above flag by:
```text
$ python -m sysconfig | grep 'no-omit-frame-pointer'
```
If no output it means that the interpreter has not been compiled with frame pointers and therefore it may not be able to show python functions in the output of perf.

<hr>

### # test profiling python program in action
```text
todo
```

<hr>

### # reference
ref: https://realpython.com/python312-perf-profiler/
ref: https://docs.python.org/3/howto/perf_profiling.html 
