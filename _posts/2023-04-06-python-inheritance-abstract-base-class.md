---
layout: post
title: "abstract base class (python)"
author: "twistfatezz"
date: 2023-04-06 22:01
categories: "2023"
tags:
  - python
  - ongoing
---

### # sample code for ABC (1)
abstract base classes(ABC): exist to be inherited, but never instantiated
```text
from abc import ABC, abstractmethod

class employee(ABC):                 # by inheriting ABC => employee is an abstract class
    def __init__(self, id, name):
        self.id = id
        self.name = name

    @abstractmethod                  # the calculate_payroll method must be overwrite by inheritance class
    def calculate_payroll(self):
        pass

hr = employee("001", "jenifer")      # abstract class cannot be instantiated
```
```txt
Traceback (most recent call last):
File "./test.py", line xx, in <module>
hr = employee("001", "jenifer")
TypeError: Can't instantiate abstract class employee with abstract methods calculate_payroll
```

but an exception is that: class inheriting ABC without abstract method can be instantiated
```text
from abc import ABC, abstractmethod

class employee(ABC):                 # by inheriting ABC => employee is an abstract class
    def __init__(self, id, name):
        self.id = id
        self.name = name

    def calculate_payroll(self):
        pass

hr = employee("001", "jenifer")      # obj successfully created
```

<hr>

### # sample code for ABC (2)
1) defined abstractmethod but not inherited from ABC => can be instantiated.
```text
from abc import ABC, abstractmethod
class Animal:
    @abstractmethod
    def eat(self):
        pass

ant = Animal()                       # obj successfully created
```
2) inherited from ABC but no abstractmethod defined => can be instantiated.
```text
from abc import ABC, abstractmethod
class Animal(ABC):
    def eat(self):
        pass

ant = Animal()                       # obj successfully created
```
3) inherited from ABC and abstractmethod defined => a nominally uninstantiable class.
```text
from abc import ABC, abstractmethod
class Animal(ABC):
    def eat(self):
        pass

ant = Animal()                       # wrong:
```
```txt
Traceback (most recent call last):
File "./test.py", line xx, in <module>
ant = Animal()  # wrong:
TypeError: Can't instantiate abstract class Animal with abstract methods eat
```
4) trying to defeat the above "no obj instantiation" restriction in python code:
```text
from abc import ABC, abstractmethod
class Animal(ABC):
    def eat(self):
        pass

Animal.__abstractmethods__           # output: frozenset({'eat'})
Animal.__abstractmethods__ = set()   # setup Animal abstract method as normal set
Animal.__abstractmethods__           # output: set()
ant = Animal()                       # try instantiate the class, successfully instantiated
```

<hr>

### # why use "abstract base class" in python? <br>
```txt
ref: https://stackoverflow.com/a/19328146
ref: https://stackoverflow.com/a/3571946
```


