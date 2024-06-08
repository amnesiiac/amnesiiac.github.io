---
layout: post
title: "inter-process communication: signals (linux, ipc)"
author: "melon"
date: 2024-01-10 22:45
categories: "2024"
tags:
  - linux
  - ipc
---

this article covers low-level signal for ipc.

<hr>

### # mechanism of signals
a signal interrupts an executing program, and in this sense, communicates with it.
most signals can be either ignored or handled, however SIGKILL and SIGSTOP are two
exceptions.

```text
#include <stdio.h>
#include <signal.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

void graceful(int signum){                                       // handler function
    printf("-> child confirm a signal received: %i\n", signum);
    puts("-> child about to terminate gracefully...");
    sleep(1);
    puts("-> child terminates now...");
    _exit(0);                                                    // fast-track notify the parent without cleanup
}

void set_handler(){                                              // setup handler
    struct sigaction current;
    sigemptyset(&current.sa_mask);                               // clear the signal set
    current.sa_flags = 0;                                        // enables setting sa_handler, not sa_action
    current.sa_handler = graceful;                               // specify a handler function
    sigaction(SIGTERM, &current, NULL);                          // register handler for SIGTERM
}

void child_code(){
    set_handler();
    while(1){                                                    // loop until interrupted by parent
        sleep(1);
        puts("-> child just woke up, but going back to sleep.");
    }
}

void parent_code(pid_t cpid){
    puts("parent sleeping for a time...");
    sleep(5);
    if(kill(cpid, SIGTERM) == -1){                               // wait 5s then send term to child
        perror("kill");
        exit(-1);
    }
    wait(NULL);                                                  // wait for all child to terminate
    puts("my child terminated, about to exit myself...");        // so that child wont be zombie proc
}

int main(){
    pid_t pid = fork();                                          // parent create child
    if(pid < 0){
        perror("fork");
        return -1;
    }
    if(pid == 0){                                                // child
        child_code();
    }
    else{                                                        // parent
        parent_code(pid);
    }
    return 0;                                                    // normal exit
}
```

compile & execute the program:
```text
$ gcc -o s sig.c && ./s
parent sleeping for a time...
-> child just woke up, but going back to sleep.
-> child just woke up, but going back to sleep.
-> child just woke up, but going back to sleep.
-> child just woke up, but going back to sleep.
-> child confirming received signal: 15
-> child about to terminate gracefully...
-> child terminating now...
my child terminated, about to exit myself...
```

using signals for ipc is indeed a minimalist approach, but a tried-and-true one at that.
ipc through signals clearly belongs in the ipc toolbox.
