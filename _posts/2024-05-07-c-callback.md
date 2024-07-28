---
layout: post
title: "callback function in c (c, callback)"
author: "melon"
date: 2024-05-10 21:49
categories: "2024"
tags:
  - c
  - todo
---

callback is a function that is stored as a reference and is called by another function, so as to
go back to the original abstraction layer.

a) synchronous or blocking callback: function that accept a callback param, is designed to
invoke the callback before finishing its job.  
b) asynchronous, non-blocking or deferred callback: function that accept a callback param, is designed
to temporarily store & handover the callback so that the callback can be called after function return.

ref: https://en.wikipedia.org/wiki/Callback_%28computer_programming%29

<hr>

### # c callback toy code

```text
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>

int stop = 0;                                          // global

void* func2(void* callback){                           // general void* to accept all func ptr type
    sleep(3);
    void (*cb)() = callback;                           // set callback
    cb();                                              // execute callback
    return NULL;
}

void stop_func1(){                                     // callback func definition
    stop = 1;
}

void func1(){
    pthread_t thread;
    pthread_create(&thread, NULL, func2, stop_func1);  // create thread to execute func2
    while(!stop){                                      // event loop
        printf("func1 is running...\n");
        sleep(1);
    }
    pthread_join(thread, NULL);                        // wait for thread end
    printf("func1 has stopped.\n");
}

int main(){
    func1();
    return 0;
}
```

compile & execute the above program:

```text
$ gcc -o out test.c -lpthread

$ ./out
func1 is running...
func1 is running...
func1 is running...
func1 is running...
func1 has stopped.
```

<hr>

### # screen saver app (based on callback technique)
