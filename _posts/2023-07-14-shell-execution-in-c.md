---
layout: post
title: "ways to execute shell cmd & script in c program (c, shell)"
author: "melon"
date: 2023-07-03 22:14
categories: "2024"
tags:
  - c
  - shell
---

method-1: execute shell script & cmd in child process, and the parent process wait for child termination &
return the pid/status of the execution.

```text
// $ gcc -o out x.c
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>                                          // _exit
#include <stdio.h>

void exec_shell_script(const char* cmd, int* out_status, pid_t* out_pid){
    pid_t pid = fork();
    if(pid == -1){
        perror("fork");
        return;
    }
    else if(pid == 0){                                       // child
        execl("/bin/sh", "sh", "-c", cmd, (char*) NULL);
        perror("execl");
        _exit(127);
    }
    else{                                                    // parent
        int status;
        if(waitpid(pid, &status, 0) == -1){
            perror("waitpid");
            return;
        }
        if(out_status){
            *out_status = status;
        }
        if(out_pid){
            *out_pid = pid;
        }
        return;
    }
}

int main(){
    const char* cmd = "ls";
    int status;
    pid_t pid;
    exec_shell_script(cmd, &status, &pid);
    if(!WEXITSTATUS(status)){
        printf("shell cmd succeed.\n");
        printf("pid: %d\n", pid);
        printf("status: %d\n", WEXITSTATUS(status));
    }
    else{
        printf("shell cmd failed.\n");
    }
    return 0;
}
```

test the above program:

```text
$ gcc -o out x.c && ./out
LICENSE   out     shell.c  test.c  tt.sh      tty-clock.1  ttyclock.h  x.c   y.sh
Makefile  readme  sig.c    t.sh    tty-clock  ttyclock.c   x.          x.sh
shell cmd succeed.
pid: 13277
status: 0
```

<p style="margin-bottom: 20px;"></p>

method-2: execute shell script & cmd in the pipe created by main process, return the execution result.

```text
// gcc -o out test.c
#include <stdio.h>
#include <stdlib.h>

void exec_shell_helper(const char* cmd){
    char buffer[128];
    FILE* pipe = popen(cmd, "r");
    if(!pipe){
        fprintf(stderr, "popen() failed!\n");
        return;
    }
    while(fgets(buffer, sizeof(buffer), pipe) != NULL){
        printf("%s", buffer);
    }
    pclose(pipe);
}

int main(){
    exec_shell_helper("ls -l");
    return 0;
}
```

test the above program:

```text
total 192
-rw-r--r--  1 mac  staff   1545 Jul 12 22:27 LICENSE
-rw-r--r--  1 mac  staff   1715 Jul 12 22:31 Makefile
-rwxr-xr-x  1 mac  staff     65 Jul 14 19:12 clock.sh
-rwxr-xr-x  1 mac  staff     22 Jul 14 19:07 k.sh
-rwxr-xr-x  1 mac  staff  13060 Jul 14 19:13 out
-rw-r--r--  1 mac  staff   4099 Jul 12 22:27 readme
-rw-r--r--  1 mac  staff    743 Jul 14 19:13 shell.c
-rw-r--r--  1 mac  staff    420 Jul 14 18:53 sig.c
-rw-r--r--  1 mac  staff   2906 Jul 14 18:28 test.c
-rw-r--r--  1 mac  staff   3193 Jul 12 22:27 tty-clock.1
-rw-r--r--  1 mac  staff  20738 Jul 12 22:27 ttyclock.c
-rw-r--r--  1 mac  staff   4359 Jul 12 22:27 ttyclock.h
-rw-r--r--  1 mac  staff    889 Jul 14 18:59 x.sh
-rwxr-xr-x  1 mac  staff    739 Jul 14 18:59 y.sh
```

<p style="margin-bottom: 20px;"></p>

method-3: execute shell script & cmd using system() function, return the status of execution result.

```text
// gcc -o out test.c
#include <stdio.h>
#include <stdlib.h>

int exec_shell_helper(const char* cmd){
    int status = system(cmd);
    return status;
}

int main(){
    exec_shell_helper("ls -l");
    return 0;
}
```
