---
layout: post
title: "Obj-oriented Programming (c)"
author: "twistfatezz"
date: 2022-12-10 12:05
categories: "2022"
tags:
  - c
---
### # Data Encapsulation
Animal.h: definition of the parent class.
```c
#ifndef _ANIMAL_H_
#define _ANIMAL_H_

typedef struct {
    int age;
    int weight;
} Animal; // 2 int variable encapsulized in type Animal

void Animal_ctor(Animal* this, int age, int weight); // ctor
int Animal_get_age(Animal* this); // getor
int Animal_get_weight(Animal* this); // getor

#endif
```
Animal.c: implementation of the parent member function.
```c
#include "Animal.h"

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
Main.c: for testing parent class encapsulation code.
```c
#include <stdio.h>
#include "Animal.c"

int main(){
    Animal a; Animal_ctor(&a, 1, 3); // construct a obj
    printf("age=%d, weight=%d \n", Animal_get_age(&a), Animal_get_weight(&a));
    return 0;
}
```
Memory model: the encapsulized data with relative bounded methods.
```txt
------------------------------------------
> memory model of the parent class: Animal
------------------------------------------
┌───────────────────────────────────┐
│ stack         code segment        │
│┌──────────┐  ┌───────────────────┐│
││int age   │  │Animal_ctor()      ││
││int weight│  │Animal_get_age()   ││
│└──────────┘  │Animal_get_weight()││
│ obj-a        └───────────────────┘│
└───────────────────────────────────┘
```

<hr>

### # Inheritance Between the Parent & Child Class
Dog.h: definition of the child class with part of data 'inherited'. 
```c
#ifndef _DOG_H_
#define _DOG_H_
#include 'Animal.h'

typedef struct {
    Animal parent; // obj inherited from parent class
    int legs; // add self-owned attributes
} Dog;

void Dog_ctor(Dog* this, int age, int weight, int legs);
int Dog_get_age(Dog* this);
int Dog_get_weight(Dog* this);
int Dog_get_legs(Dog* this);

#endif
```
Doc.c: implementation of the child member function.
```c
#include Dog.h

void Dog_ctor(Dog* this, int ages, int weight, int legs){
    this->age = age;
    this->weight = weight;
    this->legs = legs;
}
int Dog_get_age(Dog* this){ // for parent common attributes
    return Animal_get_age(&this->parent); // redirect to parent's method
}
int Dog_get_weight(Dog* this){
    return Animal_get_weight(&this->parent);
}
int Dog_get_legs(Dog* this){ // for child's self attributes
    return this->legs; // just return directly
}
```
Main.c: for testing inheritance feature.
```c
#include <stdio.h>
#include 'Animal.h'
#include 'Dog.h'

int main(){
    Dog d; Dog_ctor(&d, 1, 2, 4); // construct child class instance
    printf("age=%d, weight=%d, legs=%d \n", 
        Dog_get_age(&d), Dog_get_weight(&d), Dog_get_legs(&d));
    return 0;
}
```
Memory model: the encapsulized data with inherited part with relative methods.
```txt
--------------------------------------
> memory model of the child class: Dog
--------------------------------------
┌───────────────────────────────────┐
│ stack         code segment        │
│┌──────────┐  ┌───────────────────┐│
││int age   │  │Dog_ctor()         ││
││int weight│  │Dog_get_age()      ││
││──────────│  │Dog_get_weight()   ││
││int legs  │  │Dog_get_legs()     ││
│└──────────┘  └───────────────────┘│
└───────────────────────────────────┘
```

<hr>

### # Polymorphism implementation with 'virtual functon'
In c++, if a parent class defined a virtual function, then compiler will create a virtual function table in memory, items in this table are of type function pointer. A pointer to the virtual table is added in the memory model of this parent class. 

When the class is inherited by a child class, a new child virtual function table will be created, and the inherited pointer to virtual table is redirected to the child's own virtual table.

Note that implementation of virtual table may vary among compilers.
```txt
--------------------------------------
> memory model of virtual table in c++
--------------------------------------
┌──────────────────────────────────────────────────┐
│                          ┌──────────────────────┐│
│ virtual table       ┌───>│function1(params){...}││
│┌───────────────────┐│    └──────────────────────┘│
││void (*ptr_func1)()││    ┌──────────────────────┐│
││void (*ptr_func2)()│────>│function2(params){...}││
││void (*ptr_func3)()││    └──────────────────────┘│
│└───────────────────┘│    ┌──────────────────────┐│
│                     └───>│function3(params){...}││
│                          └──────────────────────┘│
└──────────────────────────────────────────────────┘
```
Animal.h: adding virtual table & pointer to virtual table.
```c
#ifndef _ANIMAL_H_
#define _ANIMAL_H_

struct Animal_vtable; // pre-declaration
typedef struct{
    struct Animal_vtable* vtable_ptr; // pointer to virtual table
    int age;
    int weight;
} Animal;

struct Animal_vtable{ // real definition
    void (*vfunc_ptr)(Animal* this); // function ptr
}

void Animal_ctor(Animal* this, int age, int weight);
void Animal_say(Animal* this); // declaration of parent member function
#endif
```
Animal.c: 
```c
#include <assert.h>
#include 'Animal.h'

static void _Animal_say(Animal* this){ // _ prefix is to identify 'virtual'
    // parent class Animal is an abstract concept, no need to implemnt/invoke
    // which is similar to c++ pure virtual class
    // implementation need to be done in child inherited one
    assert(0); // -----------> todo
}
void Animal_ctor(Animal* this, int age, int weight){
    static struct Animal_vtable vtable = {_Animal_say}; // init virtual table
    this->vtable_ptr = &vtable; // set vtable_ptr as vtable handler
    this->age = age;
    this->weight = weight;
}
void Animal_say(Animal* this){ // the polymorphism func call entrance
    this->vtable_ptr->vfunc_ptr(this); // find the vtable + find the func
}
```
Dog.h: the same with above inheritance one. 
```c
#ifndef _DOG_H_
#define _DOG_H_
#include 'Animal.h'

typedef struct {
    Animal parent; // obj inherited from parent class
    int legs; // add self-owned attributes
} Dog;

void Dog_ctor(Dog* this, int age, int weight, int legs);
int Dog_get_age(Dog* this);
int Dog_get_weight(Dog* this);
int Dog_get_legs(Dog* this);

#endif
```
Dog.c: implementation of the child class. 
```c
#include "dog.h"
// virtual function between parent/child
static void _Dog_say(Animal* this){
    printf("Dog says: wolfe, wolfe~");
}
void Dog_ctor(Dog* this, int age, int weight, int legs){
    Animal_ctor(&this->parent, age, weight); // init all inherited attributes
    static struct Dog_vtable vtable = {_Dog_say}; // init child's vtable
    this->parent.vtable_ptr = &vtable; // redirect inherited ptr
    this->legs = legs; // self-owned attributes
}
// non-virtual function (self-owned functions)
int Dog_get_age(Dog* this){
    return Animal_get_age(&this->parent);
}
int Dog_get_weight(Dog* this){
    return Animal_get_weight(&this->parent);
}
int Dog_get_legs(Dog* this){
    return this->legs;
}
```
Main.c: for testing polymorphically calling virtual functions.
```c
#include <stdio.h>
#include "dog.h"

int main(){
    Dog d; Dog_ctor(&d, 1, 3, 4); // creat a child obj
    Animal* ptr = &d; // parent's pointer hold a child obj (polymorphism)
    Animal_say(ptr); // polymorphism entrance func: invoke child vfunc
    return 0;
}
```
```txt
-----------------------------
> Animal obj a's memory model
-----------------------------
┌─────────────────────────────────────────────────────────┐
│ stack                          code segment             │
│┌─────────────────────────┐    ┌───────────────────┐     │
││Animal_vtable* vtable_ptr│    │Animal_ctor()      │     │
││int age             │    │    │Animal_get_age()   │     │
││int weight          │    │    │Animal_get_weight()│     │
│└────────────────────┼────┘    └───────────────────┘     │
│ Animal's vfunc table│               vfunc implementation│
│┌────────────────────+──────────┐   ┌───────────────┐    │
││void (*vfunc_ptr)(Animal* this)┼───┼> _Animal_say()│    │
││# other virtual func's type────┼───┼> _v_func()    │    │
│└───────────────────────────────┘   └───────────────┘    │
└─────────────────────────────────────────────────────────┘
```
```txt
--------------------------
> Dog obj d's memory model
--------------------------
┌─────────────────────────────────────────────────────────┐
│ stack                          code segment             │
│┌─────────────────────────┐    ┌───────────────────┐     │
││Animal_vtable* vtable_ptr│    │Dog_ctor()         │     │
││int age             │    │    │Dog_get_age()      │     │
││int weight          │    │    │Dog_get_weight()   │     │
││────────────────────┼────│    │Dog_get_legs()     │     │
││int legs            │    │    └───────────────────┘     │
│└────────────────────┼────┘                              │
│ Dog's vfunc table   │               vfunc implementation│
│┌────────────────────+──────────┐   ┌───────────────┐    │
││void (*vfunc_ptr)(Animal* this)┼───┼> _Dog_say()   │    │
││# other virtual func's type────┼───┼> _v_func()    │    │
│└───────────────────────────────┘   └───────────────┘    │
└─────────────────────────────────────────────────────────┘
```

<hr>

### # Reference
> https://zhuanlan.zhihu.com/p/338267632 

<hr>

### # Keyword Desc
