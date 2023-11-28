---
layout: post
title: "race condition and critical section (linux kernel architecture ch5)"
author: "melon"
date: 2023-09-09 17:45
categories: "2023"
tags:
  - linux
  - todo
---

### # introduction
A race condition occurs when multiple threads/processes try to modify the shared resource.

Operating systems use scheduling algorithms like first-come-first-serve, priority scheduling, round robin scheduling, to determine which processes get processed by the CPU in which order.

If an OS uses any preemptive scheduling algorithm, a process that is being processed by the CPU can get preempted to make way for another process. This process may continue its computing in its next CPU burst.

<hr>

### # race condition between parent/child process
```text
#include <iostream>
#include <stdio.h>    // setbuf
#include <unistd.h> 

// extern void setbuf (FILE* __restrict __stream, char* __restrict __buf) __THROW;
// by default, printf will stored the str in buf, and direct to stdout when flushing occurs
void print_unbuffered(const char* str){
    setbuf(stdout, NULL);                // make stdout stream unbuffered
    while(true){                         // print each char of the input str
        char ch = *str;
        if(ch == '\0'){
            break;
        }
        printf("%c", ch);                // alternatively, use putc
        str++;
    }
}

int main(){
    std::string parent_str;
    int limit = 10;                            // set 10/50
    for(int i=0; i<limit; i++){                // buildup string to be print in parent proc
        parent_str += "parent ";
    }
    std::string child_str = "child child child child child";
    pid_t pid = fork();                        // create child proc
    if(pid < 0){                               // err handle 
        return -1;
    }
                                               // execute different code for parent/child runtime
    if(pid){                                   // pid!=0 -> inside parent proc
        print_unbuffered(parent_str.c_str());
    }
    else{                                      // pid==0 -> inside child proc
        print_unbuffered(child_str.c_str());
    }
    return 0;
}
```
set the limit str len as 10, and run the above program, will output:
```text
parent parent parent parent parent parent parent parent parent parent child child child child child
```
no race condition is proved.  
The reason is that all of the characters of the str in parent proc are printed in one CPU burst.  
To see the mixed output due to race condition between parent/child, two ways to accomplish this:  
1) keep running the program repeatedly  
2) increase the size of the message by set the limit as 50, the program output:
```text
parent parent parent parent parent parent parent parent parent parent parent parent parent parent parent 
parent parent parent parent parent parent parent parent parent parent parent parent parent parent parent 
parent parent parent parent parent parent childparen tc hpialrde ncth ipladr ecnhti lpda rcehnitl dparent 
parent parent parent parent parent parent parent parent parent
```
and in which race condition occurs.

<hr>

### # race condition between two threads in same process
```text
#include <unistd.h>                                      // sleep
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

typedef struct{                                          // informaion shared by threads
    int i;
    int die;
} shareinfo;

typedef struct{                                          // information unique to each thread
    int id;
    shareinfo* p_si;
} info;

void* thread(void* x){                                   // thread code
    info* p_info;                                        // setup thread's unique data
    p_info = (info*) x;
    while(!p_info->p_si->die){                           // if shared data state not die (race condition)
        p_info->p_si->i = p_info->id;                    // keep writing self num to shared data i
    }
    return NULL;
}

int main(int argc, char** argv){
    pthread_t tids[2];                                   // .
    shareinfo si;                                        // shared data
    info i[2];                                           // unique data
    void* retval;                                        // .

                                                         // setup unique data (id) for info to threads
    i[0].id = 0;                                         // thread num 
    i[0].p_si = &si;                                     // set ptr to shared data
    i[1].id = 1;
    i[1].p_si = &si;

    si.die = 0;                                          // init as "not die"
    
    if(pthread_create(tids, NULL, thread, i) != 0){      // create threads and sleep
        perror("pthread_create"); exit(1);
    }
    if(pthread_create(tids+1, NULL, thread, i+1) != 0){
        perror("pthread_create"); exit(1);
    }

    sleep(2);                                            // two thread are preempting cpu to write to shared data
    si.die = 1;                                          // set shared data as "dir" to finish 2 thread
    printf("%d\n", si.i);                                // print shared data: thread num now

    if(pthread_join(tids[0], &retval) != 0){             // wait the threads finished, print the share info again
        perror("pthread_join"); exit(1);
    }
    if(pthread_join(tids[1], &retval) != 0){
        perror("pthread_join"); exit(1);
    }
    printf("%d\n", si.i);                                // print shared data: thread num after 2 thread exited

    return 0;
}
```
repititively run the above program, the result are as follows:
```txt
// 1st run
1
0
// 2nd run
0
1
// 3ed run
1
1
// 4th run
0
0
```

<hr>

### # race conditon: threads preempting cpu example
```text
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

typedef struct{                                                     // struct to hold thread info
    int id;                                                         // thread name
    int size;                                                       // 
    int iterations;                                                 // times to print the char
    char* s ;                                                       // thread str
} thread_struct;

void* infloop(void* x){                                             // thread runtime func
    thread_struct* t;                                               // resolve thread info
    t = (thread_struct*) x;
    int i, j;
    for(i=0; i<t->iterations; i++){                                 // print the logo for iteration time 
        for(j=0; j<t->size-1; j++){                                 // setup thread-specific "char"
            t->s[j] = 'A' + t->id;
        }
        t->s[j] = '\0';
        printf("Thread %d: %s\n", t->id, t->s); // print thread str
    }
    return NULL;
}

int main(int argc, char** argv){
    pthread_t* tid;                                                 // thread type
    thread_struct* t;                                               // thread resource -> as thread func param
    void* retval;
    int nthreads, size, iterations, i;
    char* s;

    if(argc != 4){
        fprintf(stderr, "usage: race nthreads stringsize iterations\n");
        exit(1);
    }

    nthreads = atoi(argv[1]);                                       // num of threads
    size = atoi(argv[2]);                                           // num of char in string
    iterations = atoi(argv[3]);                                     // iterations to print thread str

    tid = (pthread_t*) malloc(sizeof(pthread_t) * nthreads);        // malloc thread handle
    t = (thread_struct*) malloc(sizeof(thread_struct) * nthreads);  // malloc thread-specific struct
    s = (char*) malloc(sizeof(char*) * size);                       // malloc critical sections pointer

    for(i=0; i<nthreads; i++){                                      // set thread_struct & spawn threads
        t[i].id = i;
        t[i].size = size;
        t[i].iterations = iterations;
        t[i].s = s;                                                 // setup ptr to critical section for each thread
        if(pthread_create(tid+i, NULL, infloop, t+i) != 0){
            perror("pthread_create"); exit(1);
        }
    }
    for(i=0; i<nthreads; i++){                                      // wait for all thread to exit
        if(pthread_join(tid[i], &retval) != 0){
            perror("pthread_join"); exit(1);
        }
    }
    return 0;
}
```
compile using makefile to generate binary "out".  
the expected output is something like:
```txt
$ ./out 4 4 3
Thread 0: AAA
Thread 0: AAA
Thread 0: AAA
Thread 1: BBB
Thread 1: BBB
Thread 1: BBB
Thread 2: CCC
Thread 2: CCC
Thread 2: CCC
Thread 3: DDD
Thread 3: DDD
Thread 3: DDD
```
unfortunately, the output is not guaranteed, a single run gives the output:
```txt
$ ./out 4 4 3
Thread 0: DDD
Thread 0: AAA
Thread 0: AAA
Thread 1: AAA
Thread 1: BBB
Thread 1: BBB
Thread 3: BBB
Thread 3: DDD
Thread 3: DDD
Thread 2: DDD
Thread 2: CCC
Thread 2: CCC
```
that is, each thread are racing to take over cpu and do the work in infloop func.  
if we level up the iterations, which will give:
```txt
...
Thread 0: AAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
Thread 1: AAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
Thread 1: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBBBBBAAAAAAAAAAAAAAAAAAAAAAAAAA
Thread 1: BBBBBAAAAAABBBBBAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Thread 1: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBB
Thread 1: AAAAAAAAAAAAAAAAAAAAAABBBAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Thread 0: BBBBAAAAAABBBBBBBBBBBBBBBBBBBAABBAAAAAAAAAAAABBBBBBBBBBAAAAAAAAAAAAAA
Thread 1: AAAAAAAAAAAAAAAAABBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
Thread 0: BBBBBBBBBBBBBBBBBBBBBBBBBAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
...
```
the reason for "AB" appear is due to the cpu preemption between threads.  
the thread can be preempted anywhere: in the middle of for loop, in the middle of printf()...

<hr>

### # solution to race condition: mutex lock (or so-called binary semaphore)
```text
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

typedef struct{
    pthread_mutex_t *lock;                          // ptr to thread-shared mutex lock
    int id; 
    int size;
    int iterations;
    char *s;                                        // ptr to critical section data
} thread_struct;

void* infloop(void* x){
    int i, j;
    thread_struct* t;
 
    t = (thread_struct*) x;

    for(i=0; i<t->iterations; i++){
        if(pthread_mutex_lock(t->lock) != 0){       // pthread lock
            perror("Mutex lock"); exit(1);
        }
        for(j=0; j<t->size-1; j++){                 // enter & modify critical section
            t->s[j] = 'A' + t->id;
        }
        t->s[j] = '\0';
        printf("Thread %d: %s\n", t->id, t->s);
        if(pthread_mutex_unlock(t->lock) != 0){     // pthraed unlock
            perror("Mutex unlock"); exit(1);
        }
    }
    return NULL;
}

int main(int argc, char** argv){
    pthread_mutex_t lock;
    pthread_t* tid;
    thread_struct* t;
    void* retval;
    int nthreads, size, iterations, i;
    char* s;

    if(argc != 4){
        fprintf(stderr, "usage: race nthreads stringsize iterations\n");
        exit(1);
    }

    if(pthread_mutex_init(&lock, NULL) != 0){                       // init mutex lock
        perror("Mutex init"); exit(1);
    }
    nthreads = atoi(argv[1]);
    size = atoi(argv[2]);
    iterations = atoi(argv[3]);

    tid = (pthread_t*) malloc(sizeof(pthread_t) * nthreads);
    t = (thread_struct*) malloc(sizeof(thread_struct) * nthreads);
    s = (char*) malloc(sizeof(char*) * size);                       // malloc critical section

    for(i=0; i<nthreads; i++){
        t[i].id = i;                                                // setup thread specific info
        t[i].size = size;
        t[i].iterations = iterations;
        t[i].s = s;                                                 // setup ptr to critical section for each thread
        t[i].lock = &lock;
        if(pthread_create(tid+i, NULL, infloop, t+i) != 0){         // pthread_create -> work in infloop
            perror("pthread_create"); exit(1);
        }
    }
    for(i=0; i<nthreads; i++){                                      // wait for thread to exit
        if(pthread_join(tid[i], &retval) != 0){
            perror("pthread_join"); exit(1);
        }
    }
    return 0;
}
```
the threads are not interrupted when in critical section, the output is as follows:
```txt
$ ./out 4 4 1
Thread 1: BBB
Thread 0: AAA
Thread 2: CCC
Thread 3: DDD

$ ./out 4 4 3
Thread 1: BBB
Thread 1: BBB
Thread 2: CCC
Thread 2: CCC
Thread 3: DDD
Thread 3: DDD
Thread 0: AAA
Thread 0: AAA

$ ./out 2 70 100000 | grep "AB"   # none output
```
note: pthread_mutex_lock does not actively lock other threads, instead it lock the critical section data. each thread try to lock the critical section, if the data is unlocked, then the thread lock it and "enter" the section, or the thread will get blocked.  

<hr>

### # race condition attack: TOCTTOU (time of check to time of use)
ref: https://www.educative.io/answers/how-to-simulate-a-race-condition-in-c

<hr>

### # an extra example to illustrate the race condition / pthread_mutex_lock
```text
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <pthread.h>

struct shared{              // critical section
    int counter;            // shared counter
    pthread_mutex_t *lock;  // lock to protect the counter
    int usleeptime;         // microseconds that each thread will sleep after updating the counter
};

struct info{                // thread private data
    int id;                 // the thread's id
    struct shared *s;       // pointer to the shared information
};

void* share_counter(void* arg){                                                   // thread's exec func
    struct info* info;                                                            // thread's private info
    struct shared* s;                                                             // thread's shared info
    int counter;                                                                  // copy of the counter, to test
  
    info = (struct info*) arg;
    s = info->s;
  
    while(1){
        pthread_mutex_lock(s->lock);                                              // lock mutex
        s->counter++;                                                             // update counter
        counter = s->counter;
        printf("Thread: %3d - Begin - Counter %3d.\n", info->id, s->counter);     // print counter
        fflush(stdout);

        usleep(s->usleeptime);                                                    // sleep & print counter again
        printf("Thread: %3d - End   - Counter %3d.\n", info->id, s->counter);
        fflush(stdout);
  
        if(s->counter != counter){                                                // if counter is modified -> exit
            printf("Thread %d - Problem -- counter was %d, but now it's %d\n",
                   info->id, counter, s->counter);
            exit(1);
        }
        pthread_mutex_unlock(s->lock);                                            // unlock mutex
    }
    return NULL;
}

int main(int argc, char** argv){
    int nthreads;
    int usleeptime;
    pthread_t* tids;
    struct shared S;
    struct info* infos;
    int i;
  
    if(argc != 3){
        fprintf(stderr, "usage: mutex_example nthreads usleep_time\n");
        exit(1);
    }
  
    nthreads = atoi(argv[1]);
    usleeptime = atoi(argv[2]);
  
    tids = (pthread_t*) malloc(sizeof(pthread_t) * nthreads);
    infos = (struct info*) malloc(sizeof(struct info) * nthreads);
    for(i=0; i<nthreads; i++){
        infos[i].id = i;
        infos[i].s = &S;
    }

    S.counter = 0;
    S.usleeptime = usleeptime;
    S.lock = (pthread_mutex_t*) malloc(sizeof(pthread_mutex_t));

    pthread_mutex_init(S.lock, NULL);                                    // init mutex lock
  
    for(i=0; i<nthreads; i++){
        pthread_create(tids+i, NULL, share_counter, (void*) &infos[i]);  // create threads -> run the func
    }
  
    pthread_exit(NULL);                                                  // force threads to exit
}
```
compile the program, the output is as follows:
```text
$ ./out 4 10000 | head -n 20
```
```txt
Thread:   0 - Begin - Counter   1.
Thread:   0 - End   - Counter   1.
Thread:   1 - Begin - Counter   2.
Thread:   1 - End   - Counter   2.
Thread:   2 - Begin - Counter   3.
Thread:   2 - End   - Counter   3.
Thread:   3 - Begin - Counter   4.
Thread:   3 - End   - Counter   4.
Thread:   0 - Begin - Counter   5.
Thread:   0 - End   - Counter   5.
Thread:   1 - Begin - Counter   6.
Thread:   1 - End   - Counter   6.
Thread:   2 - Begin - Counter   7.
Thread:   2 - End   - Counter   7.
Thread:   3 - Begin - Counter   8.
Thread:   3 - End   - Counter   8.
Thread:   0 - Begin - Counter   9.
Thread:   0 - End   - Counter   9.
Thread:   1 - Begin - Counter  10.
Thread:   1 - End   - Counter  10.
```
commented the "pthread_mutex_lock()" and "pthread_mutex_unlock" out, run with same cmd, will give:
```txt
Thread:   3 - Begin - Counter   3.
Thread:   1 - Begin - Counter   1.
Thread:   0 - Begin - Counter   1.
Thread:   2 - Begin - Counter   2.
Thread:   3 - End   - Counter   3.
Thread:   1 - End   - Counter   3.
Thread:   0 - End   - Counter   3.
Thread:   2 - End   - Counter   3.
Thread:   3 - Begin - Counter   4.
Thread 1 - Problem -- counter was 1, but now it's 4
Thread 0 - Problem -- counter was 1, but now it's 4
Thread 2 - Problem -- counter was 2, but now it's 4
```
the counter is changed by other thread during thread try to change it, the result is non-deterministic.

<hr>

### # makefile to build and clean above test programs
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

ref: http://web.eecs.utk.edu/~jplank/plank/classes/cs360/360/notes/Thread-2-Race/lecture.html
