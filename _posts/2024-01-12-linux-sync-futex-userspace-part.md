---
layout: post
title: "syncronization: futex the userspace part (linux, sync)"
author: "melon"
date: 2024-01-12 21:25
categories: "2024"
tags:
  - linux
  - sync
---

fast userspace mutex (futex) mechanism was proposed by IBM in 2002, integrated to kernel
in 2003.
the idea of futex is make it more efficient for userspace code to sync multiple threads,
with minimal kernel involvement.

this article aims to provide a basic overview of futexes, work mechanism,
and how futex can be used to implement sync primitives in higher-level api and languages.

futexes are a very low-level feature of the linux kernel, suitable for foundational
runtime components (e.g. c/c++ standard libraries). it is extremely unlikely to use
them in application code.

<hr>

### # why introduce the futex?
common sync methods like mutex, semaphores are based on syscall
(lock/unlock, sem_post/sem_wait) to make access to critical section safe.
however, system calls are expensive for the context switch from userspace to kernel space.
and the sync primitives actually achieve no business logic other than keep code safe.

futex is based on the observation that in most cases, locks are actually not contended.

if a thread comes upon a free lock, locking it is cheap because there's high chance
no other thread is trying to lock it at the same time. thus, in this situation,
the lock can be acquired without a system call (block other threads locking),
but a cheaper atomic operations.

however, the unlikely event can happen at certain chance: another thread try take the lock
at the same time, make the atomic approach fail to work as assumed.

the actual solution (futex): sleep the waiting thread/process until the lock is actually
free or in high chance free, which need the global view of kernel to help with that.

<hr>

### # futex testcase (1)
```text
#include <errno.h>
#include <linux/futex.h>
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/shm.h>
#include <sys/syscall.h>
#include <sys/time.h>
#include <sys/wait.h>
#include <time.h>
#include <unistd.h>

struct timespec timeout = {  // timeout for futex wait/wakeup in non-block mode
    .tv_sec = 2,
    .tv_nsec = 0,
};

int futex(int* uaddr, int futex_op, int val, const struct timespec* timeout, int* uaddr2, int val3){
    return syscall(SYS_futex, uaddr, futex_op, val, timeout, uaddr2, val3);
}

void wait_on_futex_value(const char* proc, int* futex_addr, int val, int block){
    printf("[%s] waiting for the shared data at %p to become %d\n", proc, futex_addr, val);
    while(1){
        int futex_rc;
        if(block){
            futex_rc = futex(futex_addr, FUTEX_WAIT, val, NULL, NULL, 0); // block wait ret when futex word=val
        }
        else{
            futex_rc = futex(futex_addr, FUTEX_WAIT, val, &timeout, NULL, 0); // non-block wait with timeout
        }
        if(futex_rc == -1){
            if(errno != EAGAIN){
                perror("futex");
                exit(1);
            }
        }
        else if(futex_rc == 0){
            if(*futex_addr == val){
                printf("[%s] shared mem var at %p is equal %d\n", proc, futex_addr, val);
                return;
            }
        }
        else{
            abort();
        }
    }
}

void wake_futex(const char* proc, int* futex_addr, int block){
    printf("[%s] try wake up the futex\n", proc);
    while(1){
        int futex_rc;
        if(block){
            futex_rc = futex(futex_addr, FUTEX_WAKE, 1, NULL, NULL, 0); // block wakeup futex, ret if succeed
        }
        else{
            futex_rc = futex(futex_addr, FUTEX_WAKE, 1, &timeout, NULL, 0); // non-block wakeup futex with timeout
        }
        if(futex_rc == -1){
            perror("futex wake");
            exit(1);
        }
        else if(futex_rc > 0){
            printf("[%s] successfully wake the futex up\n", proc);
            return;
        }
    }
}

int main(int argc, char** argv){
    int shm_id = shmget(IPC_PRIVATE, 4096, IPC_CREAT | 0666);
    if(shm_id < 0){
        perror("shmget");
        exit(1);
    }

    int* shared_data = shmat(shm_id, NULL, 0);
    *shared_data = 0;
    int block = 0;                                                   // wait/wakeup in block or non-block with timeout

    int pid = fork();
    if(pid < 0){
        perror("fork");
        exit(1);
    }
    else if(pid == 0){
        wait_on_futex_value("chd", shared_data, 0xA, block);         // wait futex word = 0xA

        *shared_data = 0xB;                                          // set futex word = 0xB
        printf("[%s] set futex word as %d\n", "chd", *shared_data);

        wake_futex("chd", shared_data, block);                       // wakeup futex
    }
    else{
        *shared_data = 0xA;                                          // set futex word = 0xA
        printf("[%s] set futex word as %d\n", "par", *shared_data);

        wake_futex("par", shared_data, block);                       // wakeup futex
        wait_on_futex_value("par", shared_data, 0xB, block);         // wait futex word = 0xB

        wait(NULL);                                                  // block parent until any of its child is finished
        shmdt(shared_data);
    }

    return 0;
}
```

set block=1/0 for futex syscall, compile & execute the above program, the result are same:
```text
$ gcc -o out test.c && ./out
[par] set futex word as 10
[par] try wake up the futex
[chd] waiting for the shared data at 0x7f297a4fe000 to become 10
[par] successfully wake the futex up
[par] waiting for the shared data at 0x7f297a4fe000 to become 11
[chd] shared mem var at 0x7f297a4fe000 is equal 10
[chd] set futex word as 11
[chd] try wake up the futex
[chd] successfully wake the futex up
[par] shared mem var at 0x7f297a4fe000 is equal 11
```
the timeout for futex is set as 2s, normally the wakeup part wont take more than 2s,
result in no timeout err in non-blocking mode.

a pipeline graph to illustrate what is going on:
```txt
                           wait futex                 print futex        wakeup futex
                chd    futex(FUTEX_WAIT)   wakeup    *shared_data      futex(FUTEX_WAKE)
               ┌────────────────────────────>+───────────────────────────────>┬───────────┐
        par    │                             |                                |           │
   ───────────>┴────────────────────────────>┴───────────────────────────────>+──────────>┴──────────> end
              fork   set futex word     wakeup futex     wait futex         wakeup     wait(NULL)
                    *shared_data=0xA  futex(FUTEX_WAKE)  futex(FUTEX_WAIT)
```

<hr>

### # futex testcase (2)
```text
#include <errno.h>
#include <linux/futex.h>
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/shm.h>
#include <sys/syscall.h>
#include <sys/time.h>
#include <sys/wait.h>
#include <time.h>
#include <unistd.h>

struct timespec timeout = {
    .tv_sec = 2,
    .tv_nsec = 0,
};

int futex(int* uaddr, int futex_op, int val, const struct timespec* timeout, int* uaddr2, int val3){
    return syscall(SYS_futex, uaddr, futex_op, val, timeout, uaddr2, val3);
}

void wait_on_futex_value(const char* proc, int* futex_addr, int val, int block){
    printf("[%s] waiting for the shared data at %p to become %d\n", proc, futex_addr, val);
    while(1){
        int futex_rc;
        if(block){
            futex_rc = futex(futex_addr, FUTEX_WAIT, val, NULL, NULL, 0);      // block wait ret when futex word=val
        }
        else{
            futex_rc = futex(futex_addr, FUTEX_WAIT, val, &timeout, NULL, 0);  // non-block wait with timeout
        }
        if(futex_rc == -1){
            if(errno != EAGAIN){
                perror("futex");
                exit(1);
            }
        }
        else if(futex_rc == 0){
            if(*futex_addr == val){
                printf("[%s] shared mem var at %p is equal %d\n", proc, futex_addr, val);
                return;
            }
        }
        else{
            abort();
        }
    }
}

void wake_futex(const char* proc, int* futex_addr, int block){
    printf("[%s] try wake up the futex\n", proc);
    while(1){
        int futex_rc;
        if(block){
            futex_rc = futex(futex_addr, FUTEX_WAKE, 1, NULL, NULL, 0);      // block wakeup futex, ret if succeed
        }
        else{
            futex_rc = futex(futex_addr, FUTEX_WAKE, 1, &timeout, NULL, 0);  // non-block wakeup futex with timeout
        }
        if(futex_rc == -1){
            perror("futex wake");
            exit(1);
        }
        else if(futex_rc > 0){
            printf("[%s] successfully wake the futex up\n", proc);
            return;
        }
    }
}

void sleepzzz(const char* proc, int count){
    int i;
    for(i=count; i>0; i--){
        printf("[%s] count down before wait up the futex: %d\n", proc, i);
        sleep(1);
    }
}
int main(int argc, char** argv){
    int shm_id = shmget(IPC_PRIVATE, 4096, IPC_CREAT | 0666);
    if(shm_id < 0){
        perror("shmget");
        exit(1);
    }

    int* shared_data = shmat(shm_id, NULL, 0);
    *shared_data = 0;
    int block = 1;                                                   // wait/wakeup in blocking mode

    int pid = fork();
    if(pid < 0){
        perror("fork");
        exit(1);
    }
    else if(pid == 0){
        wait_on_futex_value("chd", shared_data, 0xA, block);         // wait futex word = 0xA
        *shared_data = 0xB;                                          // set futex word = 0xB
        printf("[%s] set futex word as %d\n", "chd", *shared_data);
        sleepzzz("chd", 4);
        wake_futex("chd", shared_data, block);                       // wakeup futex
    }
    else{
        *shared_data = 0xA;                                          // set futex word = 0xA
        printf("[%s] set futex word as %d\n", "par", *shared_data);
        sleepzzz("par", 4);
        wake_futex("par", shared_data, block);                       // wakeup futex
        wait_on_futex_value("par", shared_data, 0xB, block);         // wait futex word = 0xB

        wait(NULL);                                                  // block parent until any of its child is finished
        shmdt(shared_data);
    }

    return 0;
}

```

compile & execute the above program in blocking mode, set interval as 4s between
setup futex word and the wakeup action, the blocking behavior can be observed:
```text
$ gcc -o out test.c && ./out
[par] set futex word as 10
[par] count down before wait up the futex: 4                       <--- 4s before actually wakeup the futex
[chd] waiting for the shared data at 0x7f6ee35bd000 to become 10   <--- the child is blocking waiting for a wakeup
[par] count down before wait up the futex: 3
[par] count down before wait up the futex: 2
[par] count down before wait up the futex: 1
[par] try wake up the futex
[par] successfully wake the futex up
[par] waiting for the shared data at 0x7f6ee35bd000 to become 11   <--- the parent is blocking waiting for a wakeup
[chd] shared mem var at 0x7f6ee35bd000 is equal 10
[chd] set futex word as 11
[chd] count down before wait up the futex: 4
[chd] count down before wait up the futex: 3
[chd] count down before wait up the futex: 2
[chd] count down before wait up the futex: 1
[chd] try wake up the futex
[chd] successfully wake the futex up
[par] shared mem var at 0x7f6ee35bd000 is equal 11
```

a pipeline graph to illustrate what is going on:
```txt
                       block wait futex               print futex        wakeup futex
                chd    futex(FUTEX_WAIT)   wakeup    *shared_data      futex(FUTEX_WAKE)
               ┌────────────────────────────>+───────────────4───3───2───1───>┬───────────┐
        par    │                             |                                |           │
   ───────────>┴─────────────4───3───2───1──>┴───────────────────────────────>+──────────>┴──────────> end
              fork   set futex word     wakeup futex     block wait futex   wakeup     wait(NULL)
                    *shared_data=0xA  futex(FUTEX_WAKE)  futex(FUTEX_WAIT)
```

<hr>

### # futex testcase (3)
```text
#include <errno.h>
#include <linux/futex.h>
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/shm.h>
#include <sys/syscall.h>
#include <sys/time.h>
#include <sys/wait.h>
#include <time.h>
#include <unistd.h>

// set timeout for the futex wait/wakeup operation in non-blocking mode
struct timespec timeout = {
    .tv_sec = 2,
    .tv_nsec = 0,
};

int futex(int* uaddr, int futex_op, int val, const struct timespec* timeout, int* uaddr2, int val3){
    return syscall(SYS_futex, uaddr, futex_op, val, timeout, uaddr2, val3);
}

void wait_on_futex_value(const char* proc, int* futex_addr, int val, int block){
    printf("[%s] waiting for the shared data at %p to become %d\n", proc, futex_addr, val);
    while(1){
        int futex_rc;
        if(block){
            futex_rc = futex(futex_addr, FUTEX_WAIT, val, NULL, NULL, 0);      // block wait ret when futex word=val
        }
        else{
            futex_rc = futex(futex_addr, FUTEX_WAIT, val, &timeout, NULL, 0);  // non-block wait with timeout
        }
        if(futex_rc == -1){
            if(errno != EAGAIN){
                printf("[%s] wait for futex wakeup timeout\n", proc);
            }
        }
        else if(futex_rc == 0){
            if(*futex_addr == val){
                printf("[%s] shared mem var at %p is equal %d\n", proc, futex_addr, val);
                return;
            }
        }
        else{
            abort();
        }
    }
}

void wake_futex(const char* proc, int* futex_addr, int block){
    printf("[%s] try wake up the futex\n", proc);
    while(1){
        int futex_rc;
        if(block){
            futex_rc = futex(futex_addr, FUTEX_WAKE, 1, NULL, NULL, 0);      // block wakeup futex, ret if succeed
        }
        else{
            futex_rc = futex(futex_addr, FUTEX_WAKE, 1, &timeout, NULL, 0);  // non-block wakeup futex with timeout
        }
        if(futex_rc == -1){
            perror("futex wake");
            exit(1);
        }
        else if(futex_rc > 0){
            printf("[%s] successfully wake the futex up\n", proc);
            return;
        }
    }
}

void sleepzzz(const char* proc, int count){
    int i;
    for(i=count; i>0; i--){
        printf("[%s] count down before wait up the futex: %d\n", proc, i);
        sleep(1);
    }
}
int main(int argc, char** argv){
    int shm_id = shmget(IPC_PRIVATE, 4096, IPC_CREAT | 0666);
    if(shm_id < 0){
        perror("shmget");
        exit(1);
    }

    int* shared_data = shmat(shm_id, NULL, 0);
    *shared_data = 0;
    int block = 0;                                                   // wait/wakeup in block or non-block with timeout

    int pid = fork();
    if(pid < 0){
        perror("fork");
        exit(1);
    }
    else if(pid == 0){
        wait_on_futex_value("chd", shared_data, 0xA, block);         // wait futex word = 0xA
        *shared_data = 0xB;                                          // set futex word = 0xB
        printf("[%s] set futex word as %d\n", "chd", *shared_data);
        sleepzzz("chd", 4);
        wake_futex("chd", shared_data, block);                       // wakeup futex
    }
    else{
        *shared_data = 0xA;                                          // set futex word = 0xA
        printf("[%s] set futex word as %d\n", "par", *shared_data);
        sleepzzz("par", 4);
        wake_futex("par", shared_data, block);                       // wakeup futex
        wait_on_futex_value("par", shared_data, 0xB, block);         // wait futex word = 0xB

        wait(NULL);                                                  // block parent until any of its child is finished
        shmdt(shared_data);
    }

    return 0;
}
```

compile & execute the program in non-blocking mode, set 4s between setup futex word and
wakeup action, the timeout err print can be observed:
```text
[par] set futex word as 10
[par] count down before wait up the futex: 4                      <--- 4s before actual wakeup the futex
[chd] waiting for the shared data at 0x7fa45d63e000 to become 10  <--- non-block waiting for the wakeup
[par] count down before wait up the futex: 3
[chd] wait for futex wakeup timeout                               <--- error return for wait for futex timeout
[par] count down before wait up the futex: 2
[par] count down before wait up the futex: 1
[chd] wait for futex wakeup timeout                               <--- error return for wait for futex timeout
[par] try wake up the futex
[par] successfully wake the futex up
[par] waiting for the shared data at 0x7fa45d63e000 to become 11  <--- non-block waiting for the wakeup
[chd] shared mem var at 0x7fa45d63e000 is equal 10
[chd] set futex word as 11
[chd] count down before wait up the futex: 4
[chd] count down before wait up the futex: 3
[par] wait for futex wakeup timeout                               <--- error return for wait for futex timeout
[chd] count down before wait up the futex: 2
[chd] count down before wait up the futex: 1
[par] wait for futex wakeup timeout                               <--- error return for wait for futex timeout
[chd] try wake up the futex
[chd] successfully wake the futex up
[par] shared mem var at 0x7fa45d63e000 is equal 11
```

a pipeline graph to illustrate what is going on:
```txt
                      non-block wait futex
                      futex(FUTEX_WAIT)              print futex           wakeup futex
               child  * is timeout err     wakeup    *shared_data          futex(FUTEX_WAKE)
               ┌─────────────────*───2s──*──>+───────────────4───3───2───1───>┬───────────────┐
      parent   │                             |                                |               │
   ───────────>┴─────────────4───3───2───1──>┴───────────────────*───2s──*───>+──────────────>┴──────────> end
              fork   set futex word     wakeup futex     * is timeout err   wakeup        wait(NULL)
                    *shared_data=0xA  futex(FUTEX_WAKE)  non-block wait futex
                                                         futex(FUTEX_WAIT)
```

<hr>

### # explanation for wait(NULL)
wait(NULL) will block the parent proc until any of its child finished.

if child terminates before the parent reaches wait(NULL),
then the child process becomes a zombie until its parent waits on it and
released it from memory.

if the parent finishes firstly before child temination and does not wait(NULL),
then the child process becomes an orphan and then becomes a child of init.
the init will take care of the child proc's temination & resource release, including
remove the process entry out of the process table.
