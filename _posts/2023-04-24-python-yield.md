---
layout: post
title: "yield() method (python)"
author: "twistfatezz"
date: 2023-04-24 23:18
categories: "2023"
tags:
  - python
---

### # yield method
```python
#!/usr/bin/env python3
# ongoing...
def test(n):
    a = 0
    b = 1
    i = 0  # the loop index
    while i < n:
        yield b         # hang on yield, then return the expression value(default: None)
        a, b = b, a+b   # if called by next(), then continue the execution from the 'last hanged' yield
        # print(f'debug: {i} - {a} - {b}')
        i += 1
```

#### # case-1: next with yield basic usage
```python
if __name__ == '__main__':
    # test-case-1: using generator with next
    f = test(3)
    print(f"next-1: {next(f)}")  # b=1
    print(f"next-2: {next(f)}")  # a=1, b=1;
    print(f"next-3: {next(f)}")  # a=1, b=2;
    # print(f"next-4: {next(f)}")  # raise exception, can be handle by for loop
```
```txt
next-1: 1
next-2: 1
next-3: 2
Traceback (most recent call last):
    File "./yield.py", line 16, in <module>
    print(f"next-4: {next(f)}")
StopIteration
```

#### # case-2: using generator in for loop
```python
f = test(4)
for item in f:     # for loop will catch the StopIteration exception and exit
    print(item)
```
```txt
1
1
2
4
```
