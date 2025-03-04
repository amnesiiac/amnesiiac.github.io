---
layout: post
title: "design pattern: singleton pattern (gof)"
author: "melon"
date: 2023-11-20 22:30
categories: "2023"
tags:
  - design pattern
---

$ 1 what is thread safety?  
in programs where multiple threads share data and are executed in parallel,
thread-safe code will ensure that each thread can be executed correctly
by the synchronization mechanism to avoid data pollution.

$ 2 how to ensure thread safety?  
@1 add lock for shared resources, to ensure shared variable are only taken up by one thread each time.  
@2 provide exclusive resources for each thread, rather than shared one.
e.g. thread_local can alleviate race condition by maintaining exclusive variable for each thread,
a cpp example illustraing this is as:

```text
#include <iostream>
#include <mutex>
#include <string>
#include <thread>

using std::cout; using std::endl;

thread_local unsigned int rage = 1;                            // thread-local var
std::mutex cout_mutex;                                         // mutex
 
void increase_rage(const std::string &thread_name){
    ++rage;                                                    // modify out of mutex lock is okay for thread-local var
    std::lock_guard<std::mutex> lock(cout_mutex);              // lock the mutex
    cout << "rage counter for " << thread_name << ": " << rage << endl;
}
 
int main(){
    std::thread a(increase_rage, "a"), b(increase_rage, "b");  // create thread with associated func
    {
        std::lock_guard<std::mutex> lock(cout_mutex);          // lock the mutex for this separate scope
        cout << "rage counter for main: " << rage << endl;     // ensure following code block exec exclusively
    }
    a.join();                                                  // wait for thread a finishing
    b.join();                                                  // wait for thraed b finishing
}
```

the output can be like:

```text
rage counter for main: 1
rage counter for b: 2
rage counter for a: 2
```

examine the usage of brackets arround std::lock_guard in main, try get rid of it by:

```text
#include <iostream>
#include <mutex>
#include <string>
#include <thread>

using std::cout; using std::endl;

thread_local unsigned int rage = 1;                            // thread-local var
std::mutex cout_mutex;                                         // mutex
 
void increase_rage(const std::string &thread_name){
    ++rage;                                                    // modify out of mutex lock is okay for thread-local var
    cout_mutex.lock();                                         // lock
    cout << "rage counter for " << thread_name << ": " << rage << endl;
    cout_mutex.unlock();                                       // unlock
}
 
int main(){
    std::thread a(increase_rage, "a"), b(increase_rage, "b");  // create thread with associated func
 
    cout_mutex.lock();                                         // lock
    cout << "rage counter for main: " << rage << endl;         // ensure following code block exec exclusively
    cout_mutex.unlock();                                       // unlcok
 
    a.join();                                                  // wait for thread a finishing
    b.join();                                                  // wait for thraed b finishing
}
```

that is, the bracket arround std::lock_guard is like the with statement in python, which
enable automatically releasing lock the end of the scope.

◆ why we need singleton pattern?  
sometimes it is important to ensure that there is only one instance living of certain classes.
e.g. operating system can only have one system clock; there existed multiple printing tasks
in the printer system, but only one is for executing.

<hr>

### # singleton pattern
$ 1 definition of singleton  
@1 a singleton can only have one instance.  
@2 a singleton must create the instance by itself.  
@3 a singleton must maintain method for exposing the instance to outside.

$ 2 categories of singleton  
the singleton can be classified into lazy mode and hungry mode.
@1 lazy mode: there's no instance existed in thread context, the instance is only created
when a thread need to; extra effort is needed to make it thread-safe.  
@2 hungry mode: init the instance at the thread context, when a thread need it,
just use the instance in the thread context; naturally thread-safe method.

$ 3 feature of singleton  
@1 ctor and dtor are private, prevent construct & destory instance out of class context.  
@2 copy-ctor and value-assign-ctor are private, prevent instance creation by them out of class.  
@3 a static method to access the instance, which is accessible out of the class.

$ 4 application scenarios of lazy & hungry mode  
lazy singleton: suitable to be applied in low-access traffic scenarios; the local static version is
preferred among other lock/unlock methods.  
hungry singleton: suitable to be applied in high-access traffic scenarios.

<hr>

### # lazy singleton pattern (thread-unsafe)

```text
#include <iostream>
#include <pthread.h>

using std::cout; using std::endl;

class SingleInstance{
private:
    static SingleInstance* ptr_single_instance;                  // static member: no this
private:
    SingleInstance();                                            // ctor
    ~SingleInstance();                                           // dtor
    SingleInstance(const SingleInstance&);                       // copy ctor
    const SingleInstance& operator=(const SingleInstance&);      // assign op
public:
    static SingleInstance* get_instance();                       // static instance access api
    static void delete_instance();                               // static release instance resource 
    void print();
};

SingleInstance* SingleInstance::ptr_single_instance = nullptr;   // static data member must init ouside

SingleInstance* SingleInstance::get_instance(){                  // thread-unsafe creation without lock
                                                                 // multiple threads judge ptr=nullptr concurrently
    if(ptr_single_instance == nullptr){                          // if ptr to instance is null (no instance created)
        ptr_single_instance = new(std::nothrow) SingleInstance;  // no throw err if allocation failed (by ctor)
    }
    return ptr_single_instance;
}

void SingleInstance::delete_instance(){                          // destory instance by dtor
    if(ptr_single_instance){                                     // if instance exist, destory it
        delete ptr_single_instance;
        ptr_single_instance = nullptr;
    }
}

void SingleInstance::print(){                                    // print info
    cout << "instance memory address: " << this << endl;
}

SingleInstance::SingleInstance(){                                // ctor
    cout << "invoking ctor" << endl;
}

SingleInstance::~SingleInstance(){                               // dtor
    cout << "invoking dtor" << endl;
}
```

the lazy singleton class usecase, which introduce memory leakage in concurrency:

```text
void* printhello(void *thread_id_p){
    pthread_detach(pthread_self());                       // thread start as detached mode (no care of its end state)

    int thread_id = *((int*)thread_id_p);                 // type conversion: void* -> int* -> int
    cout << "thread id: [" << thread_id << "]" << endl;
    SingleInstance::get_instance()->print();              // create singleton instance & print addr

    pthread_exit(nullptr);                                // thread end (os is responsible for resource releasing)
}

const int NUM_THREADS = 5;                                // use const int rather than #define

int main(){
    pthread_t threads[NUM_THREADS] = {0};                                            // thread_t arr
    int index[NUM_THREADS] = {0};                                                    // index arr
    int ret = 0; int i = 0;

    cout << "main function start" << endl;
    for(int i=0; i<NUM_THREADS; ++i){                                                // 5 concurrenct th
        cout << "main(): create threads: [" << i << "]" << endl;
        index[i] = i;
        ret = pthread_create(&threads[i], nullptr, printhello, (void*)&(index[i]));  // int -> void*
        if(ret){                                                                     // err check
            cout << "error: cannot create threads" << endl;
            exit(-1);
        }
    }
    SingleInstance::delete_instance();                    // intend to release singleton resource, but 2 was created
    cout << "main function end" << endl;

    return 0;
}
```

possible output of above program can be:

```text
main function start
main(): create threads: [0]
main(): create threads: [1]
main(): create threads: [2]
thread id: [main(): create threads: [3]0]
thread id: [2]
invoking ctor
instance memory address: 0x7fcbd4405a80    // 1
invoking ctor
instance memory address: 0x7fcbd4504080    // 2
thread id: [
1]
instance memory address: 0x7fcbd4504080    // 3 (addr same as 2)
thread id: [3]
instance memory address: 0x7fcbd4504080    // 4 (addr same as 2)
main(): create threads: [4]
invoking dtor
main function end
thread id: [4]
invoking ctor
instance memory address: 0x7fcbd4504080    // 5 (addr same as 2)
```

the above program intend to create a singleton, but typically 2 instances are created
(only got 2 kinds of addr), obj creation from other threads is failed with std::bad_alloc,
but the exception is handled by std::nothrow version of new.
finally, the program try release the singleton resource, leading to memory leak.

$ 1 why lazy mode singleton is not thread-safe?  
if the concurrently running thread access the ptr==nullptr judging at the same time,
then some of them might think as no singleton created yet, leading to duplicate obj creation.

$ 2 how to avoid resource leaking?  
RAII (resource aquisition is initialization) can avoid memory leak in the above program:
when creating objects, the ctor is responsible for resource aquisition, e.g.
allocating memory, opening files, connecting to a network, etc.
when destorying objects, the dtor is responsible for resource release, e.g.
during normal exit, exception, or early return.

another way to avoid memory leak is conducted in above program: claim the thread as
detached, thus left the recycle operation to os.

$ 3 linux pthread state: detached or joinable?  
there are 2 state of linux pthread: joinable and detached. when thread is set/created as
joinable, then no recycle of thread stack & thread id at the time of exec return or
pthread_exit() to exit. when thread is set/created as detached, the thread resouce will
be released by os when exit.

$ 4 which linker option to use for managing threads: pthread or lpthread?  
both can be used for compilation of above program, but lpthread is only worked for linker,
while the pthread can worked on both pre-processor & linker. thus the pthread is recommended.

the makefile for the above program can be:

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

<hr>

### # lazy singleton pattern with lock (thread-safe)
singleton with lock class definition:

```text
#include <iostream>
#include <pthread.h>
#include <mutex>

using std::cout; using std::endl;

class SingleInstance{
private:
    static SingleInstance* ptr_single_instance;                      // static member: no this
    static std::mutex singleton_mutex;                               // mutex
private:
    SingleInstance();                                                // ctor
    ~SingleInstance();                                               // dtor
    SingleInstance(const SingleInstance&);                           // copy ctor
    const SingleInstance& operator=(const SingleInstance&);          // assign op
public:
    static SingleInstance* get_instance();                           // static instance access api
    static void delete_instance();                                   // static release instance resource 
    void print();
};

SingleInstance* SingleInstance::ptr_single_instance = nullptr;       // static data member must init ouside
std::mutex SingleInstance::singleton_mutex;                          // invoke default ctor of std::mutex

SingleInstance* SingleInstance::get_instance(){                      // double-check-lock: avoid unnecessary locking
    if(ptr_single_instance == nullptr){                              // check before lock: no singleton, then lock
        std::unique_lock<std::mutex> lock(singleton_mutex);          // lock before operate on singleton
        if(ptr_single_instance == nullptr){                          // check after lock: no singleton, then create
            ptr_single_instance = new(std::nothrow) SingleInstance;
        }
    }
    return ptr_single_instance;                                      // unlock by std::unique_lock's dtor
}

void SingleInstance::delete_instance(){
    std::unique_lock<std::mutex> lock(singleton_mutex);              // lock before operate on singleton
    if(ptr_single_instance != nullptr){
        delete ptr_single_instance;
        ptr_single_instance = nullptr;
    }                                                                // unlock by std::unique_lock's dtor
}

void SingleInstance::print(){
    cout << "instance memory address: " << this << endl;
}

SingleInstance::SingleInstance(){
    cout << "invoking ctor" << endl;
}

SingleInstance::~SingleInstance(){
    cout << "invoking dtor" << endl;
}
```

the lazy singleton class usecase code is the same as the example above.  
possible output of above program can be:

```text
main function start
main(): create threads: [0]
main(): create threads: [1]
main(): create threads: [2]
main(): create threads: [3]
thread id: [thread id: [0main(): create threads: [4]
2]
]
invoking ctor
instance memory address: 0x7fa46dd04080
thread id: [3]
instance memory address: 0x7fa46dd04080
thread id: [1]
instance memory address: 0x7fa46dd04080
thread id: [4]
instance memory address: 0x7fa46dd04080
invoking dtor
invoking ctor
instance memory address: main function end0x7fa46dd04080
```

all the object are created with the same addr, so definitely only one instance is created.

$ 1 why the lazy mode singleton pattern with lock is thread safe?  
use double-check-lock to avoid race condition to operate on the ptr to singleton,
which ensure the "check & instance creation" is only accessed by 1 process.

$ 2 drawbacks of lazy singleton pattern with lock?  
in high concurrency scenarios, tons of threads will try check the ptr_single_instance==nullptr
at the meantime, they will pass the if clause, result in fierce lock contention, leading to
lower performance.

<hr>

### # lazy singletion pattern with local static variable (c++11 thread-safe)
using local static variable as singleton, the lazy mode class definition:

```text
#include <iostream>
#include <mutex>
#include <pthread.h>

using std::cout; using std::endl;

class SingleInstance{
private:
    SingleInstance();                                            // ctor
    ~SingleInstance();                                           // dtor
    SingleInstance(const SingleInstance&);                       // copy ctor
    const SingleInstance& operator=(const SingleInstance&);      // assign op
public:
    static SingleInstance& get_instance();                       // static method: independant of class instance
    void print();
};

SingleInstance& SingleInstance::get_instance(){                  // thread-safe (static can only declared in class)
    static SingleInstance local_static_instance;                 // local static variable as singleton
    return local_static_instance;                                // return singleton's reference
}

void SingleInstance::print(){
    cout << "instance memory address: " << this << endl;
}

SingleInstance::SingleInstance(){
    cout << "invoking ctor" << endl;
}

SingleInstance::~SingleInstance(){
    cout << "invoking dtor" << endl;
}
```

the lazy singleton class usecase code is as (no need to manually release singleton resources):
```text
void* printhello(void *thread_id_p){
    pthread_detach(pthread_self());
    int thread_id = *((int*)thread_id_p);
    cout << "thread id: [" << thread_id << "]" << endl;
    SingleInstance::get_instance().print();
    pthread_exit(nullptr);
}

const int NUM_THREADS = 5;

int main(){
    pthread_t threads[NUM_THREADS] = {0};
    int index[NUM_THREADS] = {0};
    int ret = 0; int i = 0;

    cout << "main function start" << endl;
    for(int i=0; i<NUM_THREADS; ++i){
        cout << "main(): create threads: [" << i << "]" << endl;
        index[i] = i;
        ret = pthread_create(&threads[i], nullptr, printhello, (void*)&(index[i]));
        if(ret){
            cout << "error: cannot create threads" << endl;
            exit(-1);
        }
    }
    cout << "main function end" << endl;

    return 0;
}
```

possible output of above program can be:

```text
main function start
main(): create threads: [0]
main(): create threads: [1]
main(): create threads: [2]
thread id: [0]
thread id: [thread id: [1]2
main(): create threads: [invoking ctor3]
]
instance memory address: 0x108d7b120

instance memory address: 0x108d7b120
instance memory address: 0x108d7b120
thread id: [3]
instance memory address: 0x108d7b120
main(): create threads: [4]
main function end
invoking dtor
```

static variables typically stored in data or bss segment (initialized or un-initialized).

local static variable has local scope (within get_instance function body), but its lifetime
last through the whole program. thus, no need to implement dtor for them, the os will take
care of the resource recycling.

<hr>

### # hungry singleton pattern (thread-safe)
initialized the instance variable as soon as the class got defined, the hungry mode class is as:

```text
#include <iostream>
#include <mutex>
#include <pthread.h>

using std::cout; using std::endl;

class SingleInstance{
private:
    static SingleInstance* ptr_single_instance;                  // static data member (must init outside class)
private:
    SingleInstance();                                            // ctor
    ~SingleInstance();                                           // dtor
    SingleInstance(const SingleInstance&);                       // copy ctor
    const SingleInstance& operator=(const SingleInstance&);      // assign op
public:
    static SingleInstance* get_instance();                       // static instance access api
    static void delete_instance();
    void print();
};

SingleInstance* SingleInstance::ptr_single_instance = new(std::nothrow) SingleInstance;  // init once, last till end

SingleInstance* SingleInstance::get_instance(){                  // thread safe
    return ptr_single_instance;                                  // directly return the static variable
}

void SingleInstance::delete_instance(){                          // static can only be declared in class
    if(ptr_single_instance){
        delete ptr_single_instance;                              // release obj resource
        ptr_single_instance = nullptr;                           // re-assign null value to static member
    }
}

void SingleInstance::print(){
    cout << "instance memory address: " << this << endl;
}

SingleInstance::SingleInstance(){
    cout << "invoking ctor" << endl;
}

SingleInstance::~SingleInstance(){
    cout << "invoking dtor" << endl;
}
```

the hungry singleton class usecase is as:

```text
void* printhello(void *thread_id_p){
    pthread_detach(pthread_self());
    int thread_id = *((int*)thread_id_p);
    cout << "thread id: [" << thread_id << "]" << endl;
    SingleInstance::get_instance()->print();
    pthread_exit(nullptr);
}

const int NUM_THREADS = 5;

int main(){
    pthread_t threads[NUM_THREADS] = {0};
    int index[NUM_THREADS] = {0};
    int ret = 0; int i = 0;

    cout << "main function start" << endl;
    for(int i=0; i<NUM_THREADS; ++i){
        cout << "main(): create threads: [" << i << "]" << endl;
        index[i] = i;
        ret = pthread_create(&threads[i], nullptr, printhello, (void*)&(index[i]));
        if(ret){
            cout << "error: cannot create threads" << endl;
            exit(-1);
        }
    }
    SingleInstance::delete_instance();                     // resource release
    cout << "main function end" << endl;
    return 0;
}
```

possible output of above program can be:

```text
invoking ctor
main function start
main(): create threads: [0]
main(): create threads: [1]
thread id: [0]
instance memory address: 0x7ffd15c05a80
main(): create threads: [2]
thread id: [1]
instance memory address: 0x7ffd15c05a80
main(): create threads: [3]
thread id: [2]
instance memory address: 0x7ffd15c05a80
main(): create threads: [4]
thread id: [3]
instance memory address: 0x7ffd15c05a80
invoking dtor
main function end
```

that is, all the obj returned by get_instance are located at the same address, only
one instance is actually created.

<hr>

### # how to init static data member
notes about rules to static data member definition inside a class, please ref to 
[cpp reference site](https://en.cppreference.com/w/cpp/language/static),
and search for keyword: static data members.

<hr>

### # reference
cpp design pattern(GOF) chapter 1
