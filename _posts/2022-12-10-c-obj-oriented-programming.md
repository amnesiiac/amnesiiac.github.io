---
layout: post
title: "obj-oriented programming, oop (c)"
author: "melon"
date: 2022-12-10 12:05
categories: "2022"
tags:
  - c
---

object-oriented programming (oop) is a style of programming characterized by the identification of
classes of obj closely linked with the methods.

<hr>

### # encapsulation: toy code for combine data & methods
1 animal.h: definition of the parent class.

```text
#ifndef _ANIMAL_H_
#define _ANIMAL_H_

typedef struct {                                      // 2 int variable encapsulized in type Animal
    int age;
    int weight;
} Animal;

void Animal_ctor(Animal* this, int age, int weight);  // ctor
int Animal_get_age(Animal* this);                     // member access helper
int Animal_get_weight(Animal* this);                  // member access helper

#endif
```

2 animal.c: implementation of the parent member function.

```text
#include "animal.h"

void Animal_ctor(Animal* this, int age, int weight){
    this->age = age;
    this->weight = weight;
}
int Animal_get_age(Animal* this){
    return this->age;
}
int Animal_get_weight(Animal* this){
    return this->weight;
}
```

3 main.c: main process to test the parent class encapsulation code.

```text
#include <stdio.h>
#include "animal.c"

int main(){
    Animal a; Animal_ctor(&a, 1, 3);                     // construct a obj
    printf("age=%d, weight=%d \n", Animal_get_age(&a), Animal_get_weight(&a));
    return 0;
}
```

4 memory model: the encapsulized data with relative bounded methods.

```txt
                       memory model of class: Animal
                ┌─────────────────────────────────────────┐
                │  stack           code segment           │
                │ ┌────────────┐  ┌─────────────────────┐ │
                │ │ int age    │  │ Animal_ctor()       │ │
                │ │ int weight │  │ Animal_get_age()    │ │
                │ └────────────┘  │ Animal_get_weight() │ │
                │  obj-a          └─────────────────────┘ │
                └─────────────────────────────────────────┘
```

<hr>

### # encapsulation: toy code for inheritance between the parent & child class
1 dog.h: definition of the child class with part of data 'inherited'.

```text
#ifndef _DOG_H_
#define _DOG_H_
#include 'Animal.h'

typedef struct {
    Animal parent;                                        // obj inherited from parent class
    int legs;                                             // add self-owned attributes
} Dog;

void Dog_ctor(Dog* this, int age, int weight, int legs);
int Dog_get_age(Dog* this);
int Dog_get_weight(Dog* this);
int Dog_get_legs(Dog* this);

#endif
```

2 dog.c: implementation of the child member function.

```text
#include dog.h

void Dog_ctor(Dog* this, int ages, int weight, int legs){
    this->age = age;
    this->weight = weight;
    this->legs = legs;
}

int Dog_get_age(Dog* this){                            // common attr for parent & child
    return Animal_get_age(&this->parent);              // redirect to parent's method
}

int Dog_get_weight(Dog* this){                         // common attr for parent & child
    return Animal_get_weight(&this->parent);
}

int Dog_get_legs(Dog* this){                           // child's self attributes
    return this->legs;                                 // just return directly
}
```

3 main.c: for testing inheritance feature.

```text
#include <stdio.h>
#include 'Animal.h'
#include 'dog.h'

int main(){
    Dog d; Dog_ctor(&d, 1, 2, 4);                          // setup child class instance
    printf("age=%d, weight=%d, legs=%d \n",
            Dog_get_age(&d), Dog_get_weight(&d), Dog_get_legs(&d));
    return 0;
}
```

4 memory model: the encapsulized data with inherited part with relative methods.

```txt
                  memory model of the child class: Dog
                ┌──────────────────────────────────────┐
                │   stack           code segment       │
                │ ┌────────────┐  ┌──────────────────┐ │
                │ │ int age    │  │ Dog_ctor()       │ │
                │ │ int weight │  │ Dog_get_age()    │ │
                │ ├────────────┤  │ Dog_get_weight() │ │
                │ │ int legs   │  │ Dog_get_legs()   │ │
                │ └────────────┘  └──────────────────┘ │
                └──────────────────────────────────────┘
```

<hr>

### # polymorphism feature: implementation using virtual function table
in c++, if a parent class defined a virtual function, then compiler will create a virtual function table
in memory, items in this table are of type function pointer.
a pointer to the virtual table is added in the memory model of this parent class.

when the class is inherited by a child class, a new child virtual function table will be created,
and the inherited pointer to virtual table is redirected to the child's own virtual table.

note that implementation of virtual table may vary among compilers.

```txt
                   memory model of virtual table in c++
          ┌─────────────────────────────────────────────────────┐
          │                            ┌──────────────────────┐ │
          │     virtual table     ┌───>│function1(params){...}│ │
          │ ┌───────────────────┐ │    └──────────────────────┘ │
          │ │void (*ptr_func1)()│ │    ┌──────────────────────┐ │
          │ │void (*ptr_func2)()│─┼───>│function2(params){...}│ │
          │ │void (*ptr_func3)()│ │    └──────────────────────┘ │
          │ └───────────────────┘ │    ┌──────────────────────┐ │
          │                       └───>│function3(params){...}│ │
          │                            └──────────────────────┘ │
          └─────────────────────────────────────────────────────┘
```

<hr>

### # polymophism feature: toy code for design & usage of virtual func table
1 animal.h: adding virtual table & pointer to virtual table.

```text
#ifndef _ANIMAL_H_
#define _ANIMAL_H_

struct Animal_vtable;                                 // pre-declaration
typedef struct{
    struct Animal_vtable* vtable_ptr;                 // pointer to virtual table
    int age;
    int weight;
} Animal;

struct Animal_vtable{                                 // real definition
    void (*vfunc_ptr)(Animal* this);                  // function ptr
}

void Animal_ctor(Animal* this, int age, int weight);
void Animal_say(Animal* this);                        // declaration of parent member function
#endif
```

2 animal.c: 

```text
#include <assert.h>
#include 'animal.h'

static void _Animal_say(Animal* this){                   // _ prefix to identify as virtual, no impl needed
    assert(0);                                           // a nonsence placeholder
}

void Animal_ctor(Animal* this, int age, int weight){
    static struct Animal_vtable vtable = {_Animal_say};  // init virtual table, static to ensure only 1 instance
    this->vtable_ptr = &vtable;                          // set vtable_ptr
    this->age = age;
    this->weight = weight;
}

void Animal_say(Animal* this){                           // the polymorphism func call entrance
    this->vtable_ptr->vfunc_ptr(this);                   // find the vtable, then invoke the func
}
```

3 dog.h: the same with above inheritance one. 

```text
#ifndef _DOG_H_
#define _DOG_H_
#include 'animal.h'

typedef struct {
    Animal parent;                                        // obj inherited from abstract parent class
    int legs;                                             // add self-owned attributes
} Dog;

void Dog_ctor(Dog* this, int age, int weight, int legs);
int Dog_get_age(Dog* this);
int Dog_get_weight(Dog* this);
int Dog_get_legs(Dog* this);

#endif
```

4 dog.c: implementation of the child class. 

```text
#include "dog.h"

static void _Dog_say(Animal* this){                       // virtual function between parent/child
    printf("Dog says: wolfe, wolfe~");
}
void Dog_ctor(Dog* this, int age, int weight, int legs){
    Animal_ctor(&this->parent, age, weight);              // init all inherited attributes
    static struct Dog_vtable vtable = {_Dog_say};         // init child's vtable, static to ensure only 1 instance
    this->parent.vtable_ptr = &vtable;                    // redirect inherited ptr
    this->legs = legs;                                    // self-owned attributes
}

int Dog_get_age(Dog* this){                               // non-virtual function (self-owned functions)
    return Animal_get_age(&this->parent);
}

int Dog_get_weight(Dog* this){
    return Animal_get_weight(&this->parent);
}

int Dog_get_legs(Dog* this){
    return this->legs;
}
```

5 main.c: for testing polymorphically calling virtual functions.

```text
#include <stdio.h>
#include "dog.h"

int main(){
    Dog d; Dog_ctor(&d, 1, 3, 4);                // creat a child obj
    Animal* ptr = &d;                            // parent's pointer hold a child obj (polymorphism)
    Animal_say(ptr);                             // polymorphism entrance func: invoke child vfunc
    return 0;
}
```

6 memory model of the abstract base class instance:

```txt
                        Animal obj a's memory model
        ┌──────────────────────────────────────────────────────────┐
        │  stack                            code segment           │
        │ ┌───────────────────────────┐    ┌─────────────────────┐ │
        │ │ Animal_vtable* vtable_ptr │    │ Animal_ctor()       │ │
        │ │ int age             │     │    │ Animal_get_age()    │ │
        │ │ int weight          │     │    │ Animal_get_weight() │ │
        │ └─────────────────────┼─────┘    └─────────────────────┘ │
        │  Animal's vfunc table │                 vfunc impl       │
        │ ┌─────────────────────+──────────┐   ┌───────────────┐   │
        │ │ void (*vfunc_ptr)(Animal* this)┼───┼> _Animal_say()│   │
        │ │ # other virtual func's type────┼───┼> _v_func()    │   │
        │ └────────────────────────────────┘   └───────────────┘   │
        └──────────────────────────────────────────────────────────┘
```

7 memory model of the inherited child instance:

```txt
                          Dog obj d's memory model
        ┌───────────────────────────────────────────────────────────┐
        │  stack                            code segment            │
        │ ┌───────────────────────────┐    ┌────────────────────┐   │
        │ │ Animal_vtable* vtable_ptr │    │ Dog_ctor()         │   │
        │ │ int age             │     │    │ Dog_get_age()      │   │
        │ │ int weight          │     │    │ Dog_get_weight()   │   │
        │ │─────────────────────┼─────│    │ Dog_get_legs()     │   │
        │ │ int legs            │     │    └────────────────────┘   │
        │ └─────────────────────┼─────┘                             │
        │  Dog's vfunc table    │                 vfunc impl        │
        │ ┌─────────────────────+──────────┐   ┌───────────────┐    │
        │ │ void (*vfunc_ptr)(Animal* this)┼───┼> _Dog_say()   │    │
        │ │ # other virtual func's type────┼───┼> _v_func()    │    │
        │ └────────────────────────────────┘   └───────────────┘    │
        └───────────────────────────────────────────────────────────┘
```
