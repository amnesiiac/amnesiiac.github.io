---
layout: post
title: "coroutine context switch (python)"
author: "twistfatezz"
date: 2023-04-24 23:11
categories: "2023"
tags:
  - python
---

### # yield && next() to mimic coroutine context switch
sample-1
```python
#!/usr/bin/env python3
# a program to mimic coroutine with simple task context switches(no IO) in one python process
import time

def task1():
    while True:
        yield "<A> is tired, let <B> go to work"   # context switch
        print("<A> work for a while.", end='\r')   # the task1 start here, then loops
        time.sleep(1)
        print("<A> work for a while..", end='\r')
        time.sleep(1)
        print("<A> work for a while...")
        time.sleep(1)

def task2(t):
    next(t)   # start the task1 at the line after yield
    while True:
        print("-----------------------------------")
        print("<B> work for a while.", end='\r')
        time.sleep(2)
        print("<B> work for a while..", end='\r')
        time.sleep(2)
        print("<B> work for a while...")
        time.sleep(2)
        print("<B> is tired, let <A> go to work")
        ret = t.send(None)   # context switch
        print(ret)
    t.close()

if __name__ == '__main__':
    t = task1()
    task2(t)
```

```txt
-----------------------------------
<B> work for a while...
<B> is tired, let <A> go to work
<A> work for a while...               # the context switch to task1 will start after yield
<A> is tired, let <B> go to work
-----------------------------------
<B> work for a while...
<B> is tired, let <A> go to work
<A> work for a while...
<A> is tired, let <B> go to work
-----------------------------------
<B> work for a while...
...
```

sample-2
```python
#!/usr/bin/env python3
# a program to mimic coroutine with simple task context switches(no IO) in one python process
import time

def task1():
    while True:
        yield "<A> is tired, let <B> go to work"   # context switch
        print("<A> work for a while.", end='\r')   # the task1 start here, then loops
        time.sleep(1)
        print("<A> work for a while..", end='\r')
        time.sleep(1)
        print("<A> work for a while...")
        time.sleep(1)

def task2(t):
    # next(t)   # start the task1 at the line after yield
    while True:
        print("-----------------------------------")
        print("<B> work for a while.", end='\r')
        time.sleep(2)
        print("<B> work for a while..", end='\r')
        time.sleep(2)
        print("<B> work for a while...")
        time.sleep(2)
        print("<B> is tired, let <A> go to work")
        ret = t.send(None)   # context switch
        print(ret)
    t.close()

if __name__ == '__main__':
    t = task1()
    task2(t)
```
```txt
-----------------------------------
<B> work for a while...
<B> is tired, let <A> go to work
<A> is tired, let <B> go to work    # the context switch to task1 will start from the 
----------------------------------- # yield then context switch again back to task2
<B> work for a while...
<B> is tired, let <A> go to work
<A> work for a while...
<A> is tired, let <B> go to work
-----------------------------------
<B> work for a while...
...
```


### # reference

