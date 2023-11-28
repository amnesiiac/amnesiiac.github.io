---
layout: post
title: "object/BaseException (python)"
author: "twistfatezz"
date: 2023-04-05 21:05
categories: "2023"
tags:
  - python
---

nearly all class in python inherited from object
```python
# normal class
class test():
    pass

# equivelent class with test1, for compatibility of python2
class test(object):
    pass
```
show methods of object
```python
print(dir(object))
```
```txt
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__',
 '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__',
 '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__',
 '__sizeof__', '__str__', '__subclasshook__']
```
show methods of test
```python
print(dir(test))
```
```txt
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__',
 '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__',
 '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__',
 '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__']
```

however the exception class inherited from BaseException
```python
# wrong example:
class myexception():
    pass

raise myexception()
```
```txt
Traceback (most recent call last):
File "./2.py", line 23, in <module>
raise myexception()
TypeError: exceptions must derive from BaseException
```

```python
# right example:
# to create a new error type, must derive your exception from BaseException or its derived classes
class myexception(BaseException):
    pass

raise myexception()
```
```txt
Traceback (most recent call last):
  File "./2.py", line 34, in <module>
    raise myexception()
__main__.myexception
```
