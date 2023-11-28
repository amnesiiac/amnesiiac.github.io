---
layout: post
title: "next() method (python)"
author: "twistfatezz"
date: 2023-04-24 23:08
categories: "2023"
tags:
  - python
---

next() method:
```python
#!/usr/bin/env python3
import time

def task():
    while True:
        yield "<A> is tired, let <B> go to work"
        print("<A> work for a while.", end='\r')
        time.sleep(1)
        print("<A> work for a while..", end='\r')
        time.sleep(1)
        print("<A> work for a while...")
        time.sleep(1)

if __name__ == '__main__':
    t = task()
    print("--- 1st next(t) output ---")
    print(next(t))
    print("--- 2nd next(t) output ---")
    print(next(t))
```
```txt
--- 1st next(t) output ---
<A> is tired, let <B> go to work
--- 2nd next(t) output ---
<A> work for a while...
<A> is tired, let <B> go to work
```
