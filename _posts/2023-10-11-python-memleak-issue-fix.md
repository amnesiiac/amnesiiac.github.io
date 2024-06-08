---
layout: post
title: "a memleak issus illustration code & fix (python, gc)"
author: "melon"
date: 2023-10-11 21:45
categories: "2023"
tags:
  - python
  - profiling
---

1 following code is illustrating memleak in python and the fix code for it:
```text
from filelock import FileLock

import os
import gc

gc.enable()
gc.set_debug(gc.DEBUG_LEAK)

class Leak(object):                # include a big class inside (mem leak)
    def dosomething(self):
        class Inner(object):
            # a big class
            a = range(1000000)
            pass
        return None
    
class Inner(object):               # maintain the big class outside (no leak)
    a = range(1000000)
    pass

class NoLeak(object):
    def dosomething(self):
        return None

def test(mode):
    if mode == 'leak':             # observe memleak by monitoring res in htop for this proc
        for i in range(100000):
            l = Leak()
            l.dosomething()
    else:
        for i in range(100000):
            n = NoLeak()
            n.dosomething()

if __name__ == '__main__':
    test(mode='leak')              # testcase 1: mem leak
    # test(mode='noleak')          # testcase 2: no leak
    gc.collect()
    print("\ngarbage objects:")
    cnt = 0
    for x in gc.garbage:
        cnt += 1
        if cnt > 1000:
            print("still more garbage. quitting ...")
            break
        try:
            s = str(x)
        except:
            s = 'error to string'
        print(type(x))
        if len(s) > 120: s = s[:117]+'...'
        print('obj="%s"' % s)
```

<p style="margin-bottom: 20px;"></p>

2 excute the python program by:
```text
$ python3 ./test.py >log 2>&1
```

or with log output to both terminal & file:
```text
$ python3 ./test.py 2>&1 | tee ./log
```

<p style="margin-bottom: 20px;"></p>

3 log output for mem leak testcase:
```text
gc: collectable <tuple 0x7f7bb7a729e8>
gc: collectable <type 0x243e0a8>
gc: collectable <getset_descriptor 0x7f7bb79c0750>
gc: collectable <dict 0x7f7bb7a645e8>
gc: collectable <getset_descriptor 0x7f7bb79c0798>
gc: collectable <tuple 0x7f7bb79b6c88>

...

garbage objects:
<class 'tuple'>
obj="(<class 'object'>,)"
<class 'type'>
obj="<class '__main__.Blah.something.<locals>.Inner'>"
<class 'getset_descriptor'>
obj="<attribute '__dict__' of 'Inner' objects>"
<class 'dict'>
obj="{'__module__': '__main__', 'a': range(0, 1000000), '__dict__':
<attribute '__dict__' of 'Inner' objects>, '__weakref_..."
<class 'getset_descriptor'>
obj="<attribute '__weakref__' of 'Inner' objects>"
<class 'tuple'>

...

gc: collectable <dict 0x7f7bb08be798>
gc: collectable <getset_descriptor 0x7f7bb08be828>
gc: collectable <tuple 0x7f7bb08ba548>
gc: collectable <tuple 0x7f7bb08bb470>
gc: collectable <type 0x7e56a78>
gc: collectable <getset_descriptor 0x7f7bb08be8b8>
gc: collectable <dict 0x7f7bb08be870>
gc: collectable <getset_descriptor 0x7f7bb08be900>

...

<class 'dict'>
obj="{'__module__': '__main__', 'a': range(0, 1000000), '__dict__':
<attribute '__dict__' of 'Inner' objects>, '__weakref_..."

still more garbage. quitting ...

gc: collectable <module 0x7f7bbf03f188>
gc: collectable <dict 0x7f7bbf038948>
gc: collectable <builtin_function_or_method 0x7f7bbf038ca8>
gc: collectable <builtin_function_or_method 0x7f7bbf038cf0>
gc: collectable <builtin_function_or_method 0x7f7bbf038d38>
gc: collectable <tuple 0x7f7bb7a57f98>
gc: collectable <cell 0x7f7bb7a4c438>

...
```

<p style="margin-bottom: 20px;"></p>

4 log of none memory leak testcase:
```text
garbage objects:
gc: collectable <module 0x101c44090>
gc: collectable <dict 0x101c3cf40>
gc: collectable <builtin_function_or_method 0x101c44220>
gc: collectable <builtin_function_or_method 0x101c44270>
gc: collectable <builtin_function_or_method 0x101c442c0>
gc: collectable <builtin_function_or_method 0x101c44310>
gc: collectable <builtin_function_or_method 0x101c44360>
gc: collectable <builtin_function_or_method 0x101c443b0>
gc: collectable <builtin_function_or_method 0x101c44400>
gc: collectable <builtin_function_or_method 0x101c44450>
gc: collectable <builtin_function_or_method 0x101c444a0>
gc: collectable <builtin_function_or_method 0x101c444f0>

...

gc: collectable <function 0x101c91900>
gc: collectable <function 0x101c91990>
gc: collectable <dict 0x101c8bc00>
gc: collectable <list 0x101c51800>
gc: collectable <SourceFileLoader 0x101c2efe0>

... 

gc: collectable <getset_descriptor 0x101c2a300>
gc: collectable <tuple 0x101c2ca90>
gc: collectable <tuple 0x101c2caf0>
```
