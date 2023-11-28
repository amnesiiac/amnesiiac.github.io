---
layout: post
title: "nonblocking io (linux io)"
author: "melon"
date: 2023-10-05 18:32
categories: "2023"
tags:
  - linux
  - ongoing
---

### # introduction to nonblocking io model
todo

<hr>

### # nonblocking io sample code with large data to transfer (apue)
```text
#include "apue.h"                                                     // apue header file
#include <errno.h>
#include <fcntl.h>                                                    // file flag

char buf[500000];                                                     // read up to 500kb content

int main(void){
    int     ntowrite;                                                 // num of byte to write
    int     nwrite;                                                   // num of byte writen by cur write
    char    *ptr;
    ntowrite = read(STDIN_FILENO, buf, sizeof(buf));                  // read: up to 500kb from stdin into buf
    fprintf(stderr, "read %d bytes\n", ntowrite);                     // print num of byte to read
    set_fl(STDOUT_FILENO, O_NONBLOCK);                                // set nonblocking flag (write to stdout)
    ptr = buf;
    while(ntowrite > 0){                                              // continously writing to stdout
        errno = 0;                                                    // 
        nwrite = write(STDOUT_FILENO, ptr, ntowrite);                 // write to stdout, setup nwrite
        fprintf(stderr, "nwrite = %d, errno = %d\n", nwrite, errno);  // print: nwrite, errno
        if(nwrite > 0){                                               // reset start ptr if cur write is valid
            ptr += nwrite;
            ntowrite -= nwrite;
        }
    }
    clr_fl(STDOUT_FILENO, O_NONBLOCK);                                // clear nonblocking flag
    exit(0);
}
```
download apue code and compile the above code using:
```text
$ gcc -o out test.c -I /Users/mac/c3/apue/apue.3e/include -L /Users/mac/c3/apue/apue.3e/lib -lapue
```
1) using file /etc/services as stdin, only redirect stderr to file, left stdin to terminal:
```text
$ ./out < /etc/services/ 2>err
# Network services, Internet style
#
# Note that it is presently the policy of IANA to assign a single well-known
# port number for both TCP and UDP; hence, most entries here have two entries
# even if the protocol doesn't support UDP operations.
#
# The latest IANA port assignments can be gotten from
#
#	http://www.iana.org/assignments/port-numbers
...
```
check the err file content as follows, we could see that the __write__ to console is separated into more than 9000 __write__ calls, even though only 500 write calls are actually write to terminal:
```text
$ cat err | head -n 10
read 500000 bytes
nwrite = 999, errno = 0       # successfully write to terminal, 500 write are done
nwrite = -1, errno = 35       # fail to write to terminal, raise __EAGIN__
nwrite = -1, errno = 35 
nwrite = 1001, errno = 0
nwrite = -1, errno = 35
nwrite = -1, errno = 35
nwrite = -1, errno = 35
nwrite = -1, errno = 35
nwrite = -1, errno = 35
```
The __errno = 35__ means the err is __EAGAIN__. More than 8000 EAGAIN err were raised during a nonblocking io of a file of 500kb. When stdout is terminal console, which has as limited buffer size, once the buffer is full, the write will return EAGAIN in non-blocking mode.  
The loop inside program to ask for an available write operation, called polling, is a waste of CPU time on a multiuser system. I/O multiplexing with a nonblocking descriptor is a more efficient way to do this.

2) using file /etc/services as stdin, redirect stderr to file, redirect stdin to file:
```text
$ ./out < /etc/services/ 2>err 1>info
```
check the stderr of the above test case, the stdin is a regular file, the write operation is expected to be executed once. When stdout is regular file, the file i/o is responsible for intaking the output, the buffer is enough for accepting 500kb in.
```text
$ cat err
read 500000 bytes
nwrite = 500000, errno = 0
```
check the stdout of the above test case, which is same as test case 1.
```text
$ cat info
# Network services, Internet style
#
# Note that it is presently the policy of IANA to assign a single well-known
# port number for both TCP and UDP; hence, most entries here have two entries
# even if the protocol doesn't support UDP operations.
#
# The latest IANA port assignments can be gotten from
#
#	http://www.iana.org/assignments/port-numbers
...
```
