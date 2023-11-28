---
layout: post
title: "shared_ptr analysis & implementation (cpp memory model)"
author: "melon"
date: 2023-10-01 20:24
categories: "2023"
tags:
  - cpp
---

### # basic view of shared_ptr
the model of shared ptr for the below code is like:
```txt
┌──────────────────┐
│ Rectangle(10, 5) │<----- shared_ptr<Rectangle> p1 (p1.use_count()=1)
│ length / breadth │<----- shared_ptr<Rectangle> p2 (p2.use_count()=2)
└──────────────────┘                           
```
a code using standard "shared_ptr" for depicting the above use case is as:
```text
#include <iostream>
#include <memory>
using namespace std;

class Rectangle{
    int length;
    int breadth;
public:
    Rectangle(int l, int b){
        length = l;
        breadth = b;
    }
    int area(){
        return length*breadth;
    }
};

int main(){
    shared_ptr<Rectangle> p1(new Rectangle(10, 5));  // p1 point to the mem of Rect obj
    cout << p1->area() << endl;                      // 50

    shared_ptr<Rectangle> p2;                        // p2 point to the same mem as p1
    p2 = p1;
    cout << p2->area() << endl;                      // 50
    cout << p1->area() << endl;                      // 50

    cout << P1.use_count() << endl;                  // 2: refcount
    return 0;
}
```

<hr>

### # defects of normal pointer 
1) memory leaks: the memory allocated by program using normal ptr, but never got freed, which may lead to excessive memory consumption or system crash.  
2) dangling pointers: the object is de-allocated from memory without modify the value of its pointer, which may lead to unwanted result using this ptr.  
3) wild pointers: pointers declared and allocated but never pointed to a valid object or address.  
4) data inconsistency: data stored in memory but not get updated in consistent manner.  
5) buffer overflow: a pointer is used to write data to memory address which is outside of allocated memory range.

1) a code which lead to memory limit exceed problem due to infinite creating Rect obj without release them when func exit:
```text
#include <iostream>
using namespace std;

class Rectangle{
private:
    int length;
    int breadth;
};

void fun(){
    Rectangle* p = new Rectangle();  // allocate mem & assign to local variable pointer p
}                                    // destroy p without de-allocate its incharge mem(obj)

int main(){
    while(1){                        // infinite loop
        fun();                       // continuously consuming heap memory
    }
}
```

2) A solution to overcome the above memory leak problem: self-maintained smart_ptr 
```text
#include <iostream>
using namespace std;

class SmartPtr{
    int* ptr;                        // inner ptr
public:
    explicit SmartPtr(int* p=NULL){  // ctor: use explicit keywork to avoid implicit type conversion
        ptr = p;
    }
    ~SmartPtr(){                     // dtor: release ptr related mem when ptr itself is destroyed
        delete (ptr);
    }
    int& operator*(){                // overload de-reference operation
        return *ptr;
    }
};

int main(){
    SmartPtr ptr(new int());         // setup obj
    *ptr = 20;                       // access mem
    cout << *ptr;
    return 0;                        // no need to call "delete ptr", the related mem is release as ptr destroyed
}
```

3) A solution with template trick to implement smart_ptr for all types
```text
#include <iostream>
using namespace std;

template <class T> class SmartPtr{
    T* ptr;                          // inner ptr
public:
    explicit SmartPtr(T* p = NULL){  // ctor:
        ptr = p;
    }
    ~SmartPtr(){                     // dtor:
        delete (ptr);
    }
    T& operator*(){                  // overload de-reference operation
        return *ptr;
    }
    T* operator->(){                 // overload ptr->mem operation: ptr->member as operator->()->member
        return ptr;
    }
};

int main(){
    SmartPtr<int> ptr(new int());
    *ptr = 20;
    cout << *ptr;
    return 0;
}
```

<hr>

### # shared_ptr self maintained implementation
For the usages of shared_ptr semantics inside versions before c++11, we need to implement our own smart_ptr to leverage project building.
```text
#include <iostream>
#include <string>
#include <cstring>

using namespace std;

class ReferenceCount{                                              // according to S.O.L.I.D, define it separately
private:
    int m_Count;                                                   // refcount
public:
    ReferenceCount():m_Count(0){                                   // default ctor
    }
    void increment(){                                              // refcount++
        ++(this->m_Count);
    }
    void decrement(){                                              // refcount--
        --(this->m_Count);
    }
    int getCount(){                                                // get refcount
        return (this->m_Count);
    }
};

template<typename T>                                               // template shared_ptr class
class SharedPointer{
private:
    T* mp_Class;                                                   // obj ptr
    ReferenceCount* mp_RefCount;                                   // refcount ptr to same obj share same refcount
public:
    SharedPointer():mp_Class(0), mp_RefCount(0){                   // default ctor
        cout << "calling default ctor:" << endl;
        this->mp_RefCount = new ReferenceCount();
        this->mp_RefCount->increment();
    }
    SharedPointer(T* pValue):                                      // param shared_ptr
        mp_Class(pValue), mp_RefCount(0){
        cout << "calling param ctor:" << endl;
        this->mp_RefCount = new ReferenceCount();
        this->mp_RefCount->increment();
    }
    SharedPointer(const SharedPointer<T>& other):                  // copy ctor, refcount++ rather than allocate new mem
        mp_Class(other.mp_Class), mp_RefCount(other.mp_RefCount){
        cout << "calling copy ctor:" << endl;
        cout << "prev:" << this->mp_RefCount->getCount();
        cout << endl;
        this->mp_RefCount->increment();
        cout << "next:" << this->mp_RefCount->getCount();
        cout << endl;
    }
    ~SharedPointer(){                                              // dtor
        cout << "calling dtor:" << endl;
        if(this->mp_RefCount){                                     // if shared_ptr member obj is available to be freed
            cout << "prev:" << this->mp_RefCount->getCount();
            cout << endl;
            this->mp_RefCount->decrement();                        // refcount--
            cout << "next:" << this->mp_RefCount->getCount();
            cout << endl;
            if(mp_RefCount->getCount() == 0){                      // release relate mem only if refcount = 0
                cout << "refcount=0, release memory allocated";
                cout << endl;
                if(this->mp_Class){                                // safe del allocate obj: avoid crash or dangling ptr
                    delete this->mp_Class;
                    this->mp_Class = nullptr;
                }
                if(this->mp_RefCount){                             // safe del refcount obj: avoid crash or dangling ptr
                    delete this->mp_RefCount;
                    this->mp_RefCount = nullptr;
                }
            }
        }
    }
    T& operator*(){                                                // overload dereference operator
        return *(this->mp_Class);
    }
    T* operator->(){                                               // overload member access operator
        return this->mp_Class;
    }
    // void operator delete(void* ptr){                            // overload delete op for shared_ptr -> not working
    //     ptr.~SharedPointer();
    // }
    SharedPointer<T>& operator=(const SharedPointer<T>& other){    // overload assignment operator
        if(this != &other){                                        // no assign to itself
            cout << "calling assign operator. ";
            cout << "prev:" << this->mp_RefCount->getCount();
            this->mp_RefCount->decrement();                        // *this no longer point to previous obj anymore
            cout << " next:" << this->mp_RefCount->getCount();
            cout << endl;
            if(this->mp_RefCount->getCount() == 0){                // safe del allocated obj if needed
                cout << "refcount=0, release memory allocated";
                cout << endl;
                if(this->mp_Class){
                    delete this->mp_Class;
                    this->mp_Class = 0;
                }
                if(this->mp_RefCount){                             // safe del alocated obj if needed
                    delete this->mp_RefCount;
                    this->mp_RefCount = 0;
                }
            }
            this->mp_Class = other.mp_Class;                       // reset inner pointer
            this->mp_RefCount = other.mp_RefCount;                 // share same refcount obj between *this and other
            this->mp_RefCount->increment();                        // refcount++
        }
        return *this;
    }
    void getRefcount(){                                            // check refcount
        cout << "Refcount:"<< this->mp_RefCount->getCount() << " ";
    }
};
```
some classes and functions for testing:
```text
class Student{                                                     // test helper class
    int m_RollNumber;
    string m_Name;

public:
    Student():                                                     // default ctor
        m_RollNumber(0), m_Name(""){
    }
    Student(int rollNumber, char* name){                           // param ctor
        this->m_RollNumber = rollNumber;
        this->m_Name = name;
    }
    Student(const Student& other){                                 // copy ctor
        this->m_RollNumber = other.m_RollNumber;
        this->m_Name = other.m_Name;
    }
    Student& operator=(const Student& other){                      // assign ctor
        this->m_RollNumber = other.m_RollNumber;
        this->m_Name = other.m_Name;
        return *this;
    }
    ~Student(){                                                    // dtor
        this->m_RollNumber = 0;
        this->m_Name.clear();
    }
    const string& getName() const{
        return m_Name;
    }
    void setName(const string& name){
        m_Name = name;
    }
    const int getRollNumber() const{
        return m_RollNumber;
    }
    void setRollNumber(int rollNumber){
        m_RollNumber = rollNumber;
    }
    void display(){
        cout << "Roll Number = " << this->m_RollNumber << endl;
        cout << "Name = " << this->m_Name << endl;
    }
};

void valuepass(SharedPointer<Student> s){                   // accept value of self-defined ptr
    SharedPointer<Student> k = s;
}

void refpass(SharedPointer<Student>& s){                    // accept ref of self-defined ptr
    SharedPointer<Student> k = s;
}
```
testing self-implemented "shared_ptr" main function:
```text
int main(){
    SharedPointer<Student> s1(new Student(1234, "melon"));  // create shared_ptr point to Student obj
    s1->display();
    s1.getRefcount();                                       // check

    cout << endl << endl;
    SharedPointer<Student> s2 = s1;                         // copy op test, the same as "SharedPointer<Student> s2(s1)
    s2->display();
    s2.getRefcount();                                       // check
    s1.getRefcount();

    cout << endl << endl;
    SharedPointer<Student> s3;                              // default ctor
    s3 = s2;                                                // assign operation test
    s3->display();
    s3.getRefcount();                                       // check
    s2.getRefcount();
    s1.getRefcount();

    cout << endl << endl;
    SharedPointer<Student> s4(s1);                          // copy ctor test
    s4->display();
    s4.getRefcount();                                       // check
    s3.getRefcount();
    s2.getRefcount();
    s1.getRefcount();

    cout << endl << endl;
    s4.~SharedPointer<Student>();                           // explicit call shared_ptr dtor, refcount--
    s4.getRefcount();                                       // check
    s3.getRefcount();
    s2.getRefcount();
    s1.getRefcount();

    cout << endl << endl;
    s3.~SharedPointer<Student>();                           // explicit call shared_ptr dtor, refcount--
    s4.getRefcount();                                       // check
    s3.getRefcount();
    s2.getRefcount();
    s1.getRefcount();

    cout << endl << endl;
    s2.~SharedPointer<Student>();                           // explicit call shared_ptr dtor, refcount--
    s4.getRefcount();                                       // check
    s3.getRefcount();
    s2.getRefcount();
    s1.getRefcount();

    cout << endl << endl;
    refpass(s1);                                            // pass ref to function test

    cout << endl << endl;
    valuepass(s1);                                          // pass value to function test

    cout << endl << endl;                                   // refcount=0, related mem(obj) destroyed, 
    s1.~SharedPointer<Student>();                           // cannot call getRefcount anymore
    // delete s1;                                           // the overload delete operation not working (todo)

    cout << endl << endl;
    cout << "return from main function" << endl;
    return 0;                                               // destory s1/s2/s3/s4
} 
```
output:
```txt
calling param ctor:
Roll Number = 1234
Name = melon
Refcount:1 

calling copy ctor:
prev:1
next:2
Roll Number = 1234
Name = melon
Refcount:2 Refcount:2 

calling default ctor:
calling assign operator. prev:1 next:0
refcount=0, release memory allocated
Roll Number = 1234
Name = melon
Refcount:3 Refcount:3 Refcount:3 

calling copy ctor:
prev:3
next:4
Roll Number = 1234
Name = melon
Refcount:4 Refcount:4 Refcount:4 Refcount:4 

calling dtor:
prev:4
next:3
Refcount:3 Refcount:3 Refcount:3 Refcount:3 

calling dtor:
prev:3
next:2
Refcount:2 Refcount:2 Refcount:2 Refcount:2 

calling dtor:
prev:2
next:1
Refcount:1 Refcount:1 Refcount:1 Refcount:1 

calling copy ctor:
prev:1
next:2
calling dtor:
prev:2
next:1


calling copy ctor:
prev:1
next:2
calling copy ctor:
prev:2
next:3
calling dtor:
prev:3
next:2
calling dtor:
prev:2
next:1


calling dtor:
prev:1
next:0
refcount=0, safely release memory allocated


return from main function
calling dtor:
prev:0
next:-1
calling dtor:
prev:-1
next:-2
calling dtor:
prev:-2
next:-3
calling dtor:
```
makefile:
```text
CXX=g++
CFLAGS=-g -Wall -std=c++11 -pthread -w
OUT=out

SUBDIR=$(shell ls -d */)

ROOTSRC=$(wildcard *.cpp)
ROOTOBJ=$(ROOTSRC:%.cpp=%.o)

SUBSRC=$(shell find $(SUBDIR) -name '*.cpp')
SUBOBJ=$(SUBSRC:%.cpp=%.o)

$(OUT):$(ROOTOBJ) $(SUBOBJ)
	$(CXX) $(CFLAGS) -o $@ $^
.cpp.o:
	$(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
	rm -f *.o $(OUT) $(ROOTOBJ) $(SUBOBJ)
```
