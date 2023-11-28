---
layout: post
title: "super() method (python)"
author: "twistfatezz"
date: 2023-04-04 22:07
categories: "2023"
tags:
  - python
---

base class definition
---------------------
```python
#!/usr/bin/python3
class Rectangle:
    def __init__(self, length, width):
        self.length = length
        self.width = width

    def area(self):
        return self.length * self.width

    def perimeter(self):
        return 2*self.length + 2*self.width
```

example 1: create inheritant class by super()
```python
# normal implementation
class Square1:
    def __init__(self, length):
        self.length = length

    def area(self):
        return self.length * self.length

    def perimeter(self):
        return 4 * self.length

# simplified implementation
class Square2(Rectangle):
    def __init__(self, length):
        # the super return a temp base class obj, enable calling method from base class
        super().__init__(length, length)
```

example 2: overwrite/rename parent method in child class
```python
class Cube(Square2):
    # overwrite the method derived from parent class
    def area(self):
        face_area = super().area()
        return face_area * 6

    # rename the method derived from parent class
    def surface_area(self):
        face_area = super().area()
        return face_area * 6

    # rectify the method derived from parent class
    def volume(self):
        face_area = super().area()
        return face_area * self.length
```

example 3: multi-inherance with latent method resolution order problem
```python
class Triangle:
    def __init__(self, base, height):
        self.base = base
        self.height = height

    def area(self):
        return 0.5 * self.base * self.height

# wrong code
class RightPyramid(Triangle, Square2):
    def __init__(self, base, slant_height):
        self.base = base
        self.slant_height = slant_height

    def area(self):
        base_area = super().area()
        perimeter = super().perimeter()
        return 0.5 * perimeter * self.slant_height + base_area

# correct code
class RightPyramid(Square2, Triangle):  # 1 change the order of super classes
    def __init__(self, base, slant_height):
        self.base = base
        self.slant_height = slant_height
        # 2 use self.base to init super().__length() to resist "length" not found error
        super().__init__(self.base)

    def area(self):
        base_area = super().area()
        perimeter = super().perimeter()
        return 0.5 * perimeter * self.slant_height + base_area
```

main function to calling above example definitions
```python
if __name__ == '__main__':
    # example-1
    r1 = Rectangle(1, 2)
    s1 = Square1(3)
    s2 = Square2(2)
    # __class__ return the class obj 
    # __name__ return the obj name
    print(f"{r1.__class__.__name__}.area(): {r1.area()}")
    print(f"{s1.__class__.__name__}.area(): {s1.area()}")
    print(f"{s2.__class__.__name__}.area(): {s2.area()}")

    # example-2
    c1 = Cube(3)
    print(f"{c1.__class__.__name__}.area(): {c1.area()}")
    print(f"{c1.__class__.__name__}.surface_area(): {c1.surface_area()}")
    print(f"{c1.__class__.__name__}.volume(): {c1.volume()}")

    # example-3
    p1 = RightPyramid(2, 4)
    print(f"{p1.__class__.__name__}.area(): {p1.area()}")
    # __mro__ return method resolution order of certain class
    print(f"the method resolution order(MRO) of {p1.__class__.__name__} is: {RightPyramid.__mro__}")
```
