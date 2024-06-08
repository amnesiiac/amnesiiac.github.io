---
layout: post
title: "profiling python by perf (python, profiling)"
author: "melon"
date: 2023-10-10 20:45
categories: "2023"
tags:
  - python
  - profiling
  - todo
---

perf profiling is helpful to optimize python program, however, the main problem using
perf with python app is: perf only gets information about native symbols
(names of func & procedures written in c). so the entity names of python functions
in your code will not appear in the output of perf.

since python 3.12, the interpreter can run in a special mode that allows python functions
to appear in the output of the perf.

when this mode is enabled, the interpreter will interpose a small piece of code compiled
on the fly before the execution of every python function, which will teach perf the
relationship between this piece of code and the associated python function using perf map files.

<hr>

### # requirements to enable perf code interposition for python program
1 the perf support is only available for linux on select architectures till 20231010.
check the output of following cmd to see if your system support using perf for python:
```text
$ python -m sysconfig | grep HAVE_PERF_TRAMPOLINE
HAVE_PERF_TRAMPOLINE=1
```

2 python should be compiled with certain cflag for best perf profiling results:
```text
CFLAGS="-fno-omit-frame-pointer -mno-omit-leaf-frame-pointer"
```

as this allows profilers to unwind using only the frame pointer but not on dwarf debug info.
because, the time when the code interposed to allow perf support is dynamically generated,
it doesn’t have any dwarf debugging information available yet.

check whether the python is compiled with cflags above:
```text
$ python -m sysconfig | grep 'no-omit-frame-pointer'
```

if no output it means that the interpreter has not been compiled with frame pointers and
therefore it may not be able to show python functions in the output of perf.

<hr>

### # perf profiling python program in action
```text
todo
```
