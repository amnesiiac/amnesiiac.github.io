---
layout: post
title: "memory leak (python)"
author: "melon"
date: 2023-09-22 22:13
categories: "2023"
tags:
  - python
---

### # sample code to reproduce memory leak issue in python nested class definition
code with memory leak issues
```text
import os
import gc

gc.enable()
gc.set_debug(gc.DEBUG_LEAK)

class Blah(object):
    # memory leak
    def something(self):
        # the issue class is defined inside claas Blah 
        class IssueClass(object):
            # a big class
            a = range(1000000)
            pass
        
        return None
    
def test():
    # see RES in htop for this processs to observe mem leak
    for i in range(100000):
        # if class c no invoking mem leak, then c will be destory & clean from mem
        c = Blah()
        c.something()


if __name__ == '__main__':
    test()                         # main loop
    gc.collect()                   # gc collection
    print("\ngarbage objects:")    # stat result
    cnt = 0
    for x in gc.garbage:
        cnt += 1
        if cnt > 1000:
            print("still more garbage. quitting ...")
            break
        try: s = str(x)
        except:
            s = 'error to string'
        print(type(x))
        if len(s) > 120: s = s[:117]+'...'
        print('obj="%s"' % s)
```
execute the program by:
```text
$ python3 test.py 2>&1 | tee ./logs
```
the logs are as follows:
```txt
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
obj="{'__module__': '__main__', 'a': range(0, 1000000), '__dict__': <attribute '__dict__' of 'Inner' objects>, '__weakref_..."
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
obj="{'__module__': '__main__', 'a': range(0, 1000000), '__dict__': <attribute '__dict__' of 'Inner' objects>, '__weakref_..."

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
refined version code: drag "IssueClass" out of Blah
```text
import os
import gc

gc.enable()
gc.set_debug(gc.DEBUG_LEAK)
    
class IssueClass(object):
    # a big class
    a = range(1000000)
    pass

class Blah(object):
    # no memory leak
    def something(self):
        return None

def test():
    # see RES in htop for this processs to observe mem leak
    for i in range(100000):
        # if class c no invoking mem leak, then c will be destory & clean from mem
        c = Blah()
        c.something()


if __name__ == '__main__':
    test()                         # main loop
    gc.collect()                   # gc collection
    print("\ngarbage objects:")    # stat result
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
execute the program by:
```text
$ python3 test.py 2>&1 | tee ./logs
```
the logs are as follows:
```txt
gc: collectable <function 0x7f1e4811e730>
gc: collectable <function 0x7f1e4811e7b8>
gc: collectable <function 0x7f1e4811e840>
gc: collectable <function 0x7f1e4811e8c8>
gc: collectable <function 0x7f1e4811e950>
gc: collectable <function 0x7f1e4811e9d8>
gc: collectable <function 0x7f1e4811ea60>
gc: collectable <function 0x7f1e4811eae8>

...

GARBAGE OBJECTS:
<class 'function'>
o="<function OrderedDict.__setitem__ at 0x7f1e4811e730>"
<class 'function'>
o="<function OrderedDict.__delitem__ at 0x7f1e4811e7b8>"
<class 'function'>
o="<function OrderedDict.__iter__ at 0x7f1e4811e840>"

...

gc: collectable <dict 0x7f1e4f45c1b0>
gc: collectable <ModuleSpec 0x7f1e4f4686d8>
gc: collectable <dict 0x7f1e4f47e708>
gc: collectable <module 0x7f1e4f485728>
gc: collectable <dict 0x7f1e4f487510>
gc: collectable <list 0x7f1e4f46c888>

...
```
