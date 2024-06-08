---
layout: post
title: "inter-process communication: pipes (linux, ipc)"
author: "melon"
date: 2024-01-10 20:45
categories: "2024"
tags:
  - linux
  - ipc
---

pipes come in two flavors, named and unnamed. pipes can be used either from the cmdline
or within programs. pipe is like channels that connect processes for communication, and typically
has a write end and a read end working in fifo manner.

<hr>

### # unnamed pipe: ipc between a fake writer & a fake reader
```text
$ sleep 5 | echo 'hello world'   # does not actually write or read
hello world
<5s>
$
```

1) why the reader print hello world once the program start?  
the writer does not write anything to the pipe, so the reader has nothing to read, and
wont wait on the pipe.

2) why the cmdline prompt return after 5s?  
the writer will send EOS (end-of-stream) to reader if its writing finished or the writer
process prematurely terminated, the unnamed pipe will persist until both writer & reader
terminated.

<hr>

### # unnamed pipe: ipc between parent and child process
```text
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <sys/wait.h>
#include <stdlib.h>

const char* process = NULL;

void read_to_end(int fd){
    char c;
    printf("[%s] reading from descriptor %d: ", process, fd);
    while(read(fd, &c, 1) != 0){
        printf("%c", c);
    }
}

int main(int argc, char** argv){
    process = "parent";                                      // parent ctx

    int fd[2];
    if(pipe(fd) != 0){                                       // create pipe in parent ctx
        printf("error creating pipe. %s", strerror(errno));
        exit(errno);
    }

    int pid = fork();                                        // separate parent / child
    if(pid == -1){
        printf("error creating pipe. %s", strerror(errno));
        exit(errno);
    }
    else if(pid == 0){                                       // child ctx
        process = "child";
        close(fd[1]);                                        // close the write end
        read_to_end(fd[0]);                                  // read
        close(fd[0]);                                        // del fd from child fdt, if not the last ref to open fdt
        exit(0);                                             // wont destory the open fdt
        // _exit(0)                                          // exit at once (without cleanup atexit)
    }
    else{                                                    // parent ctx
        const char* data = "data written in parent proc\n";
        printf("[%s] writing data to pipe\n", process);
        close(fd[0]);                                        // close the read end
        write(fd[1], data, strlen(data));                    // write
        close(fd[1]);
        waitpid(pid, 0, 0);                                  // wait certain pid to exit
        // wait(NULL);                                       // wait for all child exit
        exit(0);
    }
}
```
pipe syscall create an unidirectional data channel for ipc, with fd[0] & fd[1] denote
read and write end of the pipe.  
after the fork syscall, the parent & child proc each has a entry in their fdt linked
to the same entry in system-level open fdt.  
the data written to the fd[1] by parent is buffered by the kernel,
until it is read from fd[0] in child.

compile and execute the program:
```text
$ gcc -o out test.c && ./out
[parent] writing data to pipe
[child]  reading from descriptor 3: some fake data written by parent
```

about relationship of the fd & fdt & open fdt, refer to article fd & fdt & ofdt for details.

<hr>

### # named pipe: ipc between different processes
writer program (writer.c):
```text
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h> 
#include <unistd.h>
#include <time.h>
#include <stdlib.h>
#include <stdio.h>

#define MaxLoops         12000                    // outer loop
#define ChunkSize           16                    // how many written for 1 loop
#define IntsPerChunk         4                    // four 4-byte ints per chunk
#define MaxZs              250                    // max microseconds to sleep

int main(){
    const char* pipeName = "./fifoChannel";
    mkfifo(pipeName, 0666);                       // create named pipe, r/w for u/g/o
    int fd = open(pipeName, O_CREAT | O_WRONLY);  // open as write-only in block mode until reader comes
    if(fd < 0){
        return -1;
    }

    for(int i=0; i<MaxLoops; i++){                // write MaxWrites times
        for(int j=0; j<ChunkSize; j++){           // write ChunkSize bytes
            int chunk[IntsPerChunk];
            for(int k=0; k<IntsPerChunk; k++){    // get IntsPerChunk filled in each chunk
                chunk[k] = rand();
            }
            write(fd, chunk, sizeof(chunk));      // write
        }
        usleep((rand() % MaxZs) + 1);             // pause a bit for realism
    }

    close(fd);                                    // close pipe: generates an end-of-file
    unlink(pipeName);                             // unlink from the implementing file
    printf("%i ints sent to the pipe\n", MaxLoops * ChunkSize * IntsPerChunk);

    return 0;
}
```

reader program (reader.c):
```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>

unsigned is_prime(unsigned n){                         // ugly but efficiency
    if(n <= 3){
        return n > 1;
    }
    if((n%2)==0 || (n%3)==0){
        return 0;
    }
    for(unsigned i=5; (i*i)<=n; i+=6){
        if((n%i)==0 || (n%(i+2))==0){
            return 0;
        }
    }
    return 1;
}

int main(){
    const char* file = "./fifoChannel";

    int fd = open(file, O_RDONLY);                     // open named pipe fd in read only mode
    if(fd < 0){
        return -1;
    }

    unsigned count = 0;
    unsigned total = 0;
    unsigned primes_count = 0;

    while(1){
        int next;
        ssize_t count = read(fd, &next, sizeof(int));  // read int each time
        if(count == 0){                                // end of stream
            break;
        }
        else if(count == sizeof(int)){                 // read a 4-byte int value
            total++;
            if(is_prime(next)){                        // count num of prime
                primes_count++;
            }
        }
    }

    close(fd);                                         // close pipe
    unlink(file);                                      // unlink from the underlying file
    printf("received ints: %u, primes: %u\n", total, primes_count);

    return 0;
}
```

compile & execute the above program:
```text
$ gcc -o w writer.c && gcc -o r reader.c

$ ./writer            # in one terminal
<program hangs due to open pipe in block mode (default)>
768000 ints sent to the pipe

$ ./reader            # in another terminal
<reader open the fifo pipe for reading, unblock the writer proc>
received ints: 768000, primes: 37948
```

check the property of named pipe backing file created:
```text
$ ./w                 # create writer solely without reader attached 

$ ls -l fifoChannel
prw-r--r--  1 melon staff  0 May 13 11:48 fifoChannel

$ cat ./fifoChannel   # let the writer go
```

◆ background on fifo backing file open flags (apue)  
1) read-only, without O_NONBLOCK: open blocks until another process opens the FIFO for writing.  
2) write-only, without O_NONBLOCK: open blocks until another process opens the FIFO for reading.  
3) read-only, with O_NONBLOCK: open returns immediately.  
4) write-only, with O_NONBLOCK: open returns an error with errno set to ENXIO
unless another process has the FIFO open for reading.
