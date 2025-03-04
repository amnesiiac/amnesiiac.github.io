---
layout: post
title: "design pattern: factory pattern (gof)"
author: "melon"
date: 2023-11-20 21:10
categories: "2023"
tags:
  - design pattern
---

factory pattern is a creational pattern that provide an optimized way of object creation,
which is mainly used in complicated class hierarchy scenarios: each class obj creation is simple,
but complexity lies in the hierarchy dependency.

using an extra base class responsible for object creation of complex class hierarchies,
can make the construction of the object more manageable.

<hr>

### # factory pattern categories
factory pattern includes: simple factory pattern, factory method pattern,
abstract factory pattern.

<hr>

### # simple factory pattern
the uml chart for the simple factory pattern:

```txt
                              (abstract product class)
                                   ┌───────────┐
                                   │   Shoes   │
                                   ├───────────┤
                      ┌------------+ + show()  +------------┐
                      | inherited  └─────+─────┘  inherited |
                      |                  │                  |
                      |                  │                  |
                      |                  │                  |
                ┌─────┴─────┐     ┌──────┴──────┐     ┌─────┴─────┐
                │ PumaShoes │     │ AdidasShoes │     │ NikeShoes │
                ├───────────┤     ├─────────────┤     ├───────────┤ (concrete product class)
                │ + show()  │     │  + show()   │     │ + show()  │
                └─────+─────┘     └──────+──────┘     └─────+─────┘
                      |                  |                  |
                      |                  |                  |
                      | create  ┌────────┴────────┐  create |
                      └---------┤   ShoesFactory  ├---------┘
                                ├─────────────────┤
                                │ + createshoes() │
                                └─────────────────┘
                                  (factory class)
```

product class hierachy code: concrete product class inherited from a abstract base class.

```text
using std::cout;
using std::endl;

class Shoes{                                       // abstract product class
public:
    virtual ~Shoes();
    virtual void Show() = 0;
};

class NikeShoes: public Shoes{                     // concrete product class 1
public:
    void show(){
        cout << "Nike, Just do it" << endl;
    }
};

class AdidasShoes: public Shoes{                   // concrete product class 2
public:
    void show(){
        cout << "Adidas, Imposible is nothing" << endl;
    }
};

class PumaShoes: public Shoes{                     // concrete product class 3
public:
    void show(){
        cout << "Puma, Everything is possible" << endl;
    }
};
```

factory class code: including method to create concrete product obj by abc class ptr.
```text
enum ShoesType{Nike, Adidas, Puma};
class ShoesFactory{
public:
    Shoes* createshoes(ShoesType type){            // runtime multi-mophology by ptr to abc class (Shoes*)
        switch(type){
            case Nike:
                return new NikeShoes();
                break;
            case Adidas:
                return new AdidasShoes(); 
                break;
            case Puma:
                return new PumaShoes();
                break;
            default:
                return nullptr;
                break;
        }
    }
};
```

the application code of simple factory pattern:
```text
int main(){
    ShoesFactory putian;                                      // use simple factory pattern to create product obj
    Shoes* nike_shoes_ptr = putian.createshoes(Nike);         // create nike product shoes
    if(nike_shoes_ptr != nullptr){
        nike_shoes_ptr->show();
        delete nike_shoes_ptr;                                // release nike product obj resources
        nike_shoes_ptr = nullptr;                             // reset pointer
    }
    Shoes* adidas_shoes_ptr = putian.createshoes(Adidas);     // create Adidas product shoes
    ...
}
```

simple factory pattern actually solves the problem:  
@1 for a specific product class with complicated hierarchy,
the object creation for certain series of product is very confusing:
object naming scheme and the interface for object creation are of great diversity.
by adopting simple factory pattern, all objects are created by the same factory,
which can unify the object creation path and make the created objects to be manuplative.  
@2 the core idea of the factory pattern is:
the product hierarchy definition and the product obj creation are separated from each other
to make the program logic clearer.

<hr>

### # reference
cpp design pattern (gof) chapter 1
