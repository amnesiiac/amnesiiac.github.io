---
layout: post
title: "inter-process communication: shared storage (linux, ipc)"
author: "melon"
date: 2024-01-10 21:45
categories: "2024"
tags:
  - linux
  - ipc
---

linux process can synchronize with each other by sharing storage: shared files
or shared memory.

<hr>

### # shared files
consider the simplest example: one producer create & write to a file, and another
process read from the same file:

```txt

              create & write  ┌───────────┐      read
    producer ────────────────>│ disk file │<──────────────── consumer
                              └───────────┘

```

the challenge of the above scenario is the race condition: the producer & consumer
might access to the file at the same time, which will make the outcome inter-determinate.

a producer should gain an exclusive lock on the file before writing to it.
an exclusive lock can only be held by one process to avoid race condition,
no other process can access the file until the lock is released.

a consumer should gain at least a shared lock on the file before reading from it.
multiple readers can hold a shared lock at the same time, there is no reason to
prevent other consumer reading the file, so ahared lock can promotes efficiency.

```txt

              create & write  ┌───────────┐      read
    producer ────────────────>│ disk file │<──────────────── consumer
              exclusive lock  └───────────┘   shared lock

```

the producer code:
```text
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

#define FileName "data.dat" 
#define DataString "Now is the winter of our discontent\nMade glorious summer by this sun of York\n"

void report_and_exit(const char* msg){                      // exit with failure msg
    perror(msg); exit(-1);
}

int main(){
    struct flock lock;
    lock.l_type = F_WRLCK;                                  // r/w (exclusive versus shared) lock
    lock.l_whence = SEEK_SET;                               // base for seek offsets
    lock.l_start = 0;                                       // lock from the first byte
    lock.l_len = 0;                                         // lock until EOF (0)
    lock.l_pid = getpid();                                  // producer pid

    int fd;
    if((fd = open(FileName, O_RDWR | O_CREAT, 0666)) < 0){  // try file open
        report_and_exit("open failed");
    }

    if(fcntl(fd, F_SETLK, &lock) < 0){                      // try get lock: F_SETLK doesn't block, F_SETLKW does
        report_and_exit("fcntl failed to get lock");        // fail to get lock
    }
    else{
        write(fd, DataString, strlen(DataString));          // populate data file
        fprintf(stderr, "proc %d has written to data file\n", lock.l_pid);
    }

    lock.l_type = F_UNLCK;                                  // try release the lock
    if(fcntl(fd, F_SETLK, &lock) < 0){
        report_and_exit("explicit unlocking failed");
    }

    close(fd);                                              // close the file: unlock if needed
    return 0;                                               // term the proc would unlock as well
}
```

the consumer code:
```text
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

#define FileName "data.dat" 

void report_and_exit(const char* msg){
    perror(msg);
    exit(-1);
}

int main(){
    struct flock lock;
    lock.l_type = F_WRLCK;                               // read/write (exclusive) lock
    lock.l_whence = SEEK_SET;                            // base for seek offsets
    lock.l_start = 0;                                    // lock from the start 
    lock.l_len = 0;                                      // lock until EOF (0)
    lock.l_pid = getpid();                               // consumer pid

    int fd;
    if((fd = open(FileName, O_RDONLY)) < 0){             // try open file
        report_and_exit("open to read failed");
    }

    fcntl(fd, F_GETLK, &lock);                           // get r/w lock state (set lock.l_type=F_UNLCK if no writer)
    if(lock.l_type != F_UNLCK){
        report_and_exit("file is still write locked")
    }

    lock.l_type = F_RDLCK;                               // set r lock, prevent w during r, allow parallel r
    if(fcntl(fd, F_SETLK, &lock) < 0){
        report_and_exit("can't get a read-only lock");
    }

    int c;                                               // read buffer
    while(read(fd, &c, 1) > 0){                          // stil something to read (1 byte a time)
        write(STDOUT_FILENO, &c, 1);                     // write the read content to stdout (1 byte each time)
    }

    lock.l_type = F_UNLCK;                               // release the lock
    if(fcntl(fd, F_SETLK, &lock) < 0){
        report_and_exit("explicit unlocking failed");
    }

    close(fd); 
    return 0;  
}
```
the shared file’s contents could be voluminous, arbitrary bytes (e.g, a digitized movie),
which makes file sharing a really flexible ipc mechanism.

however, file access is typically slow, whether the access involves
reading or writing.

as always, programming comes with tradeoffs:
the next section, ipc through shared memory has a boost in performance.

<hr>

### # how semaphore works in synchronization mechanism
a general semaphore also is called a counting semaphore, as it has a init value
(typically 0) that can be incremented.

consider a shop that rents bicycles (100 total), with a program that clerks
use to do the rentals.
every time a bike is rented, the semaphore is incremented by 1;
every time a bike is returned, the semaphore is decremented by 1.
rentals can continue until the semaphore hits 100, then the rent program must halt
until at least one returned.

a binary semaphore is a special case requiring only two values: 0 and 1.
in this situation, a semaphore acts as a mutex: a mutual exclusion construct.

<hr>

### # shared memory
linux systems provide two separate apis for shared memory:
the legacy system-v api and the recent posix api, which should never be mixed
in a single application.

the posix api have impacts on the code portability in following scenarios:  
(1) the features of posix api are still in development and dependent on kernel version,
which lowers the code portability.  
(2) by default, the posix implements shared memory as a memory-mapped file,
moreover, shared memory in posix can be configured without a backing file,
which lowers the code portability.

following is a code example to illustrate the posix api for semaphore controled
shared mem access, which combines the benefits of memory access (speed) and
file storage (persistence).

the shmem.h code:
```text
#define ByteSize 512
#define BackingFile "/shMemEx"
#define AccessPerms 0644
#define SemaphoreName "mysemaphore"
#define MemContents "This is the way the world ends...\n"

void report_and_exit(const char* msg){
    perror(msg);
    exit(-1);
}
```

the memwriter.c code:
```text
/* compilation on macos: gcc -o w memwriter.c -lpthread */
/* compilation on linux: gcc -o w memwriter.c -lrt -lpthread */
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <semaphore.h>
#include <string.h>

#include "shmem.h"

int main(){
    int fd = shm_open(BackingFile, O_RDWR | O_CREAT, AccessPerms);  // create shared mem fd with backing file
    if(fd < 0){
        report_and_exit("can't open shared mem backing file");
    }
    ftruncate(fd, ByteSize);                                        // adjust the backing file size

    caddr_t memptr = mmap(
        NULL,                                                       // let system pick shared mem seg pos
        ByteSize,                                                   // bytes
        PROT_READ | PROT_WRITE,                                     // access protections
        MAP_SHARED,                                                 // shared mem visible to other proc
        fd,                                                         // fd
        0                                                           // offset: start at 0
    );
    if((caddr_t) -1 == memptr){                                     // mmap creation check
        report_and_exit("can't get shared mem segment");
    }
    fprintf(stderr, "shared mem address: %p [0..%d]\n", memptr, ByteSize - 1);
    fprintf(stderr, "backing file: /dev/shm%s\n", BackingFile );

    sem_t* semptr = sem_open(                                       // create semaphore to lock the shared mem
        SemaphoreName,                                              // name
        O_CREAT,                                                    // op = create
        AccessPerms,                                                // protection perms
        0                                                           // init = 0 (unavailable)
    );
    if(semptr == (void*) -1){                                       // check
        report_and_exit("create shared semaphore failed");
    }

    strcpy(memptr, MemContents);                                    // critical section: write to mem

    int num = 8;                                                    // just block reader out
    fprintf(stderr, "blocking reader out of critical section count down: ");
    while(num > 0){
        fprintf(stderr, "%d ", num);
        num--;
        sleep(1);
    }

    if(sem_post(semptr) < 0){                                       // post: semaphore+1 (make it available)
        report_and_exit("sem_post failed");
    }

    num = 8;                                                        // timeslot to allow reader in
    fprintf(stderr, "\nsafe timeslot to let reader retrieve result back: ");
    while(num > 0){
        fprintf(stderr, "%d ", num);
        num--;
        sleep(1);
    }

    fprintf(stderr, "\ncleanup started");                           // cleanups
    munmap(memptr, ByteSize);                                       // unmap the storage
    close(fd);
    shm_unlink(BackingFile);                                        // unlink from the backing file
    sem_close(semptr);                                              // close sema of this proc proactively (still in os)
    sem_unlink(SemaphoreName);                                      // destroy the semaphore in kernel
    fprintf(stderr, "\ncleanup finished");
    return 0;
}
```
when the semaphore is 0, only the memwriter can access the shared memory.
after writing, this writer process increments the semaphore,
thereby allowing the memreader to read the shared memory.

the memreader.c code:
```text
/* compilation on macos: gcc -o r memreader.c -lpthread */
/* compilation on linux: gcc -o r memreader.c -lrt -lpthread */
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <semaphore.h>
#include <string.h>

#include "shmem.h"

int main(){
    int fd = shm_open(BackingFile, O_RDWR, AccessPerms);  // r/w without create
    if(fd < 0){
        report_and_exit("can't get file descriptor");
    }

    caddr_t memptr = mmap(                                // ptr to mem
        NULL,                                             // let system pick mem seg pos
        ByteSize,                                         // bytes of shared mem
        PROT_READ | PROT_WRITE,                           // access protections
        MAP_SHARED,                                       // shared mem is visible to other proc
        fd,                                               // fd
        0                                                 // offset: start at 0
    );
    if((caddr_t) -1 == memptr){
        report_and_exit("can't access shared mem");
    }

    sem_t* semptr = sem_open(                             // semaphore as mutex
        SemaphoreName,                                    // name
        O_CREAT,                                          // create or reuse semaphore
        AccessPerms,                                      // protection perms
        0                                                 // init as 0
    );
    if(semptr == (void*) -1){                             // check
        report_and_exit("sem_open failed");
    }

    if(!sem_wait(semptr)){                                // wait: semaphore>0, then sema-1
        int i;
        for(i=0; i<strlen(MemContents); i++){
            write(STDOUT_FILENO, memptr+i, 1);            // read data from critical section & print on screen
        }
        sem_post(semptr);                                 // post: semaphore+1
    }

    sem_close(semptr);                                    // cleanups
    // sem_unlink(SemaphoreName);                         // no uncomment this, just keep the reader welcomed
    munmap(memptr, ByteSize);
    close(fd);
    unlink(BackingFile);                                  // if commented, file will persist after program exit
    return 0;
}
```

the writer output:
```text
$ ./writer
shared mem address: 0x10f9ae000 [0..511]
backing file: /dev/shm/shMemEx
blocking reader out of critical section count down: 8 7 6 5 4 3 2 1
safe timeslot to let reader retrieve result back: 8 7 6 5 4 3 2 1
cleanup started
cleanup finished
```

the reader output:

```text
$ ./reader
This is the way the world ends...   # print after the count down finished
```

◆ why we need to add sem_unlink() in writer?   
if comment out the sem_unlink(SemaphoreName) in writer, after the first run,
the reader will always be able to access the critical section without permission
from from writer post.  
sem_unlink is used to destory the semaphore in the os. after the writer execute
for the first time, the semaphore created will be persistent in os.
from then on (2nd/3rd run...), the reader will be able to access the previous posted
semaphore (>0), and access the critical section directly.

◆ why we wont add sem_unlink() in reader?   
if we uncommented the sem_unlink in reader, then during the 8s timeslot for reader to
access, only the first reader can access the section, and any more reader are blocked,
which is unwanted behavior.

◆ the relationship between shared mem & its backing file  
the memwriter and memreader programs access the shared memory only, not the backing file.
the system is responsible for synchronizing the shared memory and the backing file.

◆ sem_unlink vs sem_close  
(1) sem_close: close a semaphore, this also done when a process exits.
note: the semaphore still remains in the system. posix semaphores are kernel persistent,
which means the semaphore retains even if no process has the semaphore opened (count=0).  
(2) sem_unlink: will be removed from the system only when the reference count reaches 0,
when all proc have opened the semaphore, call sem_close or are all exited.  
ref: https://stackoverflow.com/a/21961950

◆ semaphore protection pipeline analysis
<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/ipc.pdf" width="700"/>

<hr>

### # ipc between threads (variants of above section)
(1) sem_open_api: writer & reader synchronization in linux threads code example, in which
each timeslot only 1 thread can actually access to the shared memory:
```text
/* compilation on linux: gcc -o t test.c -lpthread -lrt */
/* compilation on macos: gcc -o t test.c -lpthread -lrt */
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <fcntl.h>
#include <stdbool.h>

int shared_var;                                           // shared var

void* writer(void* arg){
    bool* flag = (bool*) arg;
    sem_t* sem = sem_open(                                // sem_open: create or access the named sema
        "testsema",
        O_CREAT,
        0644,
        1
    );
    if(sem == (void*) -1){
        printf("sem_open");
        return 0;
    }
    if(*flag){
        sem_wait(sem);                                    // sem_wait: if sema>0, sema-1 else blocking
    }

    shared_var = 10;                                      // critical section
    printf("writer set var as %d\n", shared_var);

    int num = 3;                                          // block reader out
    while(num > 0){
        printf("%d\n", num);
        num--;
        sleep(1);
    }

    if(*flag){
        sem_post(sem);                                    // sem_post: sema+1
        sem_close(sem);
    }
    return NULL;
}

void* reader(void* arg){
    bool* flag = (bool*) arg;
    sem_t* sem = sem_open(                                // create or reuse semaphore to lock the shared mem
        "testsema",
        O_CREAT,
        0644,
        1
    );
    if(sem == (void*) -1){
        printf("sem_open");
        return 0;
    }
    if(*flag){
        sem_wait(sem);
    }

    printf("reader get var as %d\n", shared_var);         // critical section

    if(*flag){
        sem_post(sem);
        sem_close(sem);
    }
    return NULL;
}

int main(){
    bool enable = true;                                   // enable semaphore or not

    sem_unlink("testsema");                               // sem_unlink clean dirt sema with same name

    pthread_t wt0, rt0, rt1, rt2;                         // thread stuff
    pthread_create(&rt0, NULL, reader, (void*) &enable);
    pthread_create(&rt1, NULL, reader, (void*) &enable);
    pthread_create(&wt0, NULL, writer, (void*) &enable);
    pthread_create(&rt2, NULL, reader, (void*) &enable);
    pthread_join(rt0, NULL);
    pthread_join(rt1, NULL);
    pthread_join(rt2, NULL);
    pthread_join(wt0, NULL);

    sem_unlink("testsema");                               // sem_unlink: garbage collect current sema
    return 0;
}
```

a typical output of the program is as:
```text
$ ./out
reader get var as 0     # readers access the shared mem ahead of writer got 0
reader get var as 0
writer set var as 10    # writer change it to 10
3
2
1
reader get var as 10    # after the block finished, reader got 10
```
the above implementation makes all thread to access the critical section in an exclusive
manner regardless of reader or writer.

however, it's reasonable to make all readers shared the access with each other,
so a possible improvements is to use a reader function specific semaphore with larger
init value (>1), which provide a looser permission.
a tighter semaphore among writer and readers is embedded inside the looser one.

ref: https://stackoverflow.com/a/78406003/10515951

(2) sema_init api: writer & reader synchronization in linux threads code example, in which
each time only 1 thread (read or write) can actually access the the critical section:
```text
/* compilation linux: gcc -o t test.c -lpthread -lrt */
/* compilation macos: gcc -o t test.c -lpthread */
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>

sem_t sem;                                         // global semaphore
int shared_var;                                    // shared var between threads

void* writer(void* arg){
    sem_wait(&sem);                                // wait: sem>0, then sema-1 (block)
    shared_var = 10;                               // critical section
    printf("writer set var as %d\n", shared_var);
    int num = 3;                                   // let reader to get in
    while(num > 0){
        printf("%d\n", num);
        num--;
        sleep(1);
    }
    sem_post(&sem);                                // post: sema+1
    return NULL;
}

void* reader(void* arg){
    sem_wait(&sem);                                // wait
    printf("reader get var as %d\n", shared_var);  // critical section
    sem_post(&sem);                                // post

    return NULL;
}

int main(){
    sem_init(&sem, 0, 1);                          // init: set perm & init value

    pthread_t writer_thread, reader_thread;
    pthread_create(&writer_thread, NULL, writer, NULL);
    pthread_create(&reader_thread, NULL, reader, NULL);
    pthread_join(writer_thread, NULL);
    pthread_join(reader_thread, NULL);

    sem_destroy(&sem);                             // destroy

    return 0;
}
```

the above program work fine on linux:
```text
writer set var as 10
3
2
1
reader get var as 10   # the reader is blocked until it acquire the semaphore
```

however, something wrong when compiling on macos:
```text
$ gcc -o t test.c -pthread
test.c:33:5: warning: 'sem_init' is deprecated [-Wdeprecated-declarations]
    sem_init(&sem, 0, 1);
test.c:41:5: warning: 'sem_destroy' is deprecated [-Wdeprecated-declarations]
    sem_destroy(&sem);
```

the program output on macos:
```text
writer set var as 10
3
reader get var as 10   # the reader break into the protection of writer thread
2
1
```
check the link for details of the deprecation warning on macos:
https://stackoverflow.com/a/27847103.  
