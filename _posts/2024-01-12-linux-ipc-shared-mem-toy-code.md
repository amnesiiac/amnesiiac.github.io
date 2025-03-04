---
layout: post
title: "share memory toy code (linux, ipc)"
author: "melon"
date: 2024-01-12 20:24
categories: "2024"
tags:
  - linux
  - ipc
---

this article will introduce a few toy code for share mem usage in ipc scenarios, covering both system-v
(legacy) & posix api (modern).

<hr>

### # posix: ipc via shared mem
$ 1 program a: create shared mem obj with name, write msg in it.

```text
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>
#include <unistd.h>

#define SHM_NAME "/my_shared_memory"
#define SHM_SIZE 256

int main() {
    int shm_fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);  // get fd for physical shared mem (name assigned)
    if(shm_fd < 0){
        perror("shm_open");
        exit(1);
    }

    ftruncate(shm_fd, SHM_SIZE);                              // set size of fd
                                                              // map physical mem represented by fd to proc virtual mem
    char* shared_memory = mmap(0, SHM_SIZE, PROT_WRITE, MAP_SHARED, shm_fd, 0);
    if(shared_memory == MAP_FAILED){
        perror("mmap");
        exit(1);
    }

    sprintf(shared_memory, "hello from process a!");          // write to shm

    munmap(shared_memory, SHM_SIZE);                          // detach process str from physical shared mem
    close(shm_fd);                                            // destory shm_fd resources, left physical shm untouched
    return 0;
}
```

$ 2 program b: access the shared mem segment by name, read from it then unlink it after usage.

```text
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>

#define SHM_NAME "/my_shared_memory"
#define SHM_SIZE 256

int main(){
    int shm_fd = shm_open(SHM_NAME, O_RDONLY, 0666);         // get fd for physical shm 
    if(shm_fd < 0){
        perror("shm_open");
        exit(1);
    }
                                                             // get ptr for proc virtual mem map from fd
    char* shared_memory = mmap(0, SHM_SIZE, PROT_READ, MAP_SHARED, shm_fd, 0);
    if(shared_memory == MAP_FAILED){
        perror("mmap");
        exit(1);
    }

    printf("read from shared memory: %s\n", shared_memory);  // print shared mem content

    sleep(10);                                               // mimic a long program run

    if(munmap(shared_memory, SHM_SIZE) == -1){               // detach proc ptr from physical shm, global refcount-1
        perror("munmap");
        exit(1);
    }
    close(shm_fd);                                           // destory fd resource with physical mem un-touched

    if(shm_unlink(SHM_NAME) == -1){                          // unlink physical shm, shm name removed globally
        perror("shm_unlink");                                // but underneath shm obj entry is only destoryed after
        exit(1);                                             // all proc detached (reference count = 0)
    }
    return 0;
}
```

$ 3 compile the above program, note: need link the rt lib to enable posix api:

```text
$ gcc -o a a.c -lrt
$ gcc -o b b.c -lrt
```

$ 4 test program basic function:

```text
$ ./a                                            // shm create, fill info
$ ./b                                            // shm open, get info, unmap, unlink
read from shared memory: hello from process a!
$ ./b                                            // once again access the shm, failed due to shm unlinked globally 
shm_open: No such file or directory
```

$ 5 inspectations on the share mem by procfs  
5.1 check procfs for active fd entries under:

```text
$ ls -l /proc/3108/fd/
0  1  2  3
```

5.2 list all active fd for program b (the shm is not unmapped or unlinked yet):

```text
$ ls -l /proc/3108/fd/0
lrwx------ 1 metung metung 64 Dec 30 14:43 /proc/3108/fd/0 -> /dev/pts/2

$ ls -l /proc/3108/fd/1
lrwx------ 1 metung metung 64 Dec 30 14:43 /proc/3108/fd/1 -> /dev/pts/2

$ ls -l /proc/3108/fd/2
lrwx------ 1 metung metung 64 Dec 30 14:43 /proc/3108/fd/2 -> /dev/pts/2

$ ls -l /proc/3108/fd/3
lr-x------ 1 metung metung 64 Dec 30 14:43 /proc/3108/fd/3 -> /dev/shm/my_shared_memory
```

5.3 check program b memory map info:

```text
$ cat /proc/3108/maps
proc virtual mem range            perm offset   dev   inode              file
00400000-00401000                 r-xp 00000000 fd:11 5819261            /repo1/metung/txt/alcoholism/mem/shm/b
00600000-00601000                 r--p 00000000 fd:11 5819261            /repo1/metung/txt/alcoholism/mem/shm/b
00601000-00602000                 rw-p 00001000 fd:11 5819261            /repo1/metung/txt/alcoholism/mem/shm/b
7f36e4d08000-7f36e4d1f000         r-xp 00000000 fd:01 396316             /usr/lib64/libpthread-2.17.so
7f36e4d1f000-7f36e4f1e000         ---p 00017000 fd:01 396316             /usr/lib64/libpthread-2.17.so
7f36e4f1e000-7f36e4f1f000         r--p 00016000 fd:01 396316             /usr/lib64/libpthread-2.17.so
7f36e4f1f000-7f36e4f20000         rw-p 00017000 fd:01 396316             /usr/lib64/libpthread-2.17.so
7f36e4f20000-7f36e4f24000         rw-p 00000000 00:00 0
7f36e4f24000-7f36e50e8000         r-xp 00000000 fd:01 396290             /usr/lib64/libc-2.17.so
7f36e50e8000-7f36e52e7000         ---p 001c4000 fd:01 396290             /usr/lib64/libc-2.17.so
7f36e52e7000-7f36e52eb000         r--p 001c3000 fd:01 396290             /usr/lib64/libc-2.17.so
7f36e52eb000-7f36e52ed000         rw-p 001c7000 fd:01 396290             /usr/lib64/libc-2.17.so
7f36e52ed000-7f36e52f2000         rw-p 00000000 00:00 0
7f36e52f2000-7f36e52f9000         r-xp 00000000 fd:01 396715             /usr/lib64/librt-2.17.so
7f36e52f9000-7f36e54f8000         ---p 00007000 fd:01 396715             /usr/lib64/librt-2.17.so
7f36e54f8000-7f36e54f9000         r--p 00006000 fd:01 396715             /usr/lib64/librt-2.17.so
7f36e54f9000-7f36e54fa000         rw-p 00007000 fd:01 396715             /usr/lib64/librt-2.17.so
7f36e54fa000-7f36e551c000         r-xp 00000000 fd:01 396687             /usr/lib64/ld-2.17.so
7f36e56f5000-7f36e56f9000         rw-p 00000000 00:00 0
7f36e5718000-7f36e5719000         rw-p 00000000 00:00 0
7f36e5719000-7f36e571a000         r--s 00000000 00:17 1910467964         /dev/shm/my_shared_memory
7f36e571a000-7f36e571b000         rw-p 00000000 00:00 0
7f36e571b000-7f36e571c000         r--p 00021000 fd:01 396687             /usr/lib64/ld-2.17.so
7f36e571c000-7f36e571d000         rw-p 00022000 fd:01 396687             /usr/lib64/ld-2.17.so
7f36e571d000-7f36e571e000         rw-p 00000000 00:00 0
7fffcd1ea000-7fffcd20b000         rw-p 00000000 00:00 0                  [stack]
7fffcd2cb000-7fffcd2ce000         r--p 00000000 00:00 0                  [vvar]
7fffcd2ce000-7fffcd2cf000         r-xp 00000000 00:00 0                  [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```

5.4 display memory map format in neat format:

```text
$ ls -l /proc/3108/map_files/
total 0
lr-------- 1 metung metung 64 Dec 30 15:22 400000-401000 -> /repo1/metung/txt/alcoholism/mem/shm/b
lr-------- 1 metung metung 64 Dec 30 15:22 600000-601000 -> /repo1/metung/txt/alcoholism/mem/shm/b
lr-------- 1 metung metung 64 Dec 30 15:22 601000-602000 -> /repo1/metung/txt/alcoholism/mem/shm/b
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e4d08000-7f36e4d1f000 -> /usr/lib64/libpthread-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e4d1f000-7f36e4f1e000 -> /usr/lib64/libpthread-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e4f1e000-7f36e4f1f000 -> /usr/lib64/libpthread-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e4f1f000-7f36e4f20000 -> /usr/lib64/libpthread-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e4f24000-7f36e50e8000 -> /usr/lib64/libc-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e50e8000-7f36e52e7000 -> /usr/lib64/libc-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e52e7000-7f36e52eb000 -> /usr/lib64/libc-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e52eb000-7f36e52ed000 -> /usr/lib64/libc-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e52f2000-7f36e52f9000 -> /usr/lib64/librt-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e52f9000-7f36e54f8000 -> /usr/lib64/librt-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e54f8000-7f36e54f9000 -> /usr/lib64/librt-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e54f9000-7f36e54fa000 -> /usr/lib64/librt-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e54fa000-7f36e551c000 -> /usr/lib64/ld-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e5719000-7f36e571a000 -> /dev/shm/my_shared_memory
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e571b000-7f36e571c000 -> /usr/lib64/ld-2.17.so
lr-------- 1 metung metung 64 Dec 30 15:22 7f36e571c000-7f36e571d000 -> /usr/lib64/ld-2.17.so
```

5.5 display all processes that linked to this shm (lsof utility):

```text
$ ./b &
[1] 36388
$ read from shared memory: hello from process a!
```

list all process attached with my_shared_memory:

```text
$ lsof 2>/dev/null | grep /dev/shm/my_shared_memory
b       36388 metung  mem    REG   0,23      256 1946386867 /dev/shm/my_shared_memory
b       36388 metung    3r   REG   0,23      256 1946386867 /dev/shm/my_shared_memory
```

list process 36388 open file infomations (resources):

```text
$ lsof -p 36388 2>/dev/null
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF       NODE NAME
b       36388 metung  cwd    DIR 253,17     4096    5881916 /repo1/metung/txt/alcoholism/mem/shm/posix
b       36388 metung  rtd    DIR  253,1     4096          2 /
b       36388 metung  txt    REG 253,17     8768    5819261 /repo1/metung/txt/alcoholism/mem/shm/posix/b
b       36388 metung  mem    REG  253,1   142144     396316 /usr/lib64/libpthread-2.17.so
b       36388 metung  mem    REG  253,1  2156592     396290 /usr/lib64/libc-2.17.so
b       36388 metung  mem    REG  253,1    43712     396715 /usr/lib64/librt-2.17.so
b       36388 metung  mem    REG  253,1   163312     396687 /usr/lib64/ld-2.17.so
b       36388 metung  mem    REG   0,23      256 1946386867 /dev/shm/my_shared_memory
b       36388 metung    0u   CHR  136,2      0t0          5 /dev/pts/2
b       36388 metung    1u   CHR  136,2      0t0          5 /dev/pts/2
b       36388 metung    2u   CHR  136,2      0t0          5 /dev/pts/2
b       36388 metung    3r   REG   0,23      256 1946386867 /dev/shm/my_shared_memory
```

<hr>

### # system-v: ipc via shared mem
$ 1 program a: create shm segment by key, write to it, wait for program b signal, cleanup shm & exit.

```text
#include <sys/types.h>
#include <sys/shm.h>
#include <sys/ipc.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define SHMSIZE 100

int main(){
    int   shmid;
    key_t key;
    char* shm;
    char* s;

    key = 9876;
    shmid = shmget(key, SHMSIZE, IPC_CREAT|0666);        // create a shm obj indentified by shmid
    if(shmid < 0){
        printf("%s", strerror(errno));
        perror("error in shared memory get statement");
        exit(1);
    }

    shm = shmat(shmid, NULL, 0);                         // attach proc virtual mem range to shm obj, with ptr returned
    if(shm == (char*)-1){
        perror("error in shared memory attachment");
        exit(1);
    }
    memcpy(shm, "hello world", 11);                      // write to shm
    s = shm;
    s += 11;

    while(*shm != '*'){                                  // wait for end signal from program 2
        sleep(1);
    }

    if(shmctl(shmid, IPC_RMID, NULL) == -1){             // cleanup
        perror("shmctl: remove shm obj failed");
        exit(1);
    }
    return 0;
}
```

$ 2 program b: access shm by key, print out content inside, signal program a by writing to shm.

```text
#include <sys/types.h>
#include <sys/shm.h>
#include <sys/ipc.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define SHMSIZE 100

int main(){
    int   shmid;
    key_t key;
    char* shm;
    char* s;

    key = 9876;
    shmid = shmget(key, SHMSIZE, 0666);                  // get a shm obj identified by shmid
    if(shmid < 0){
        printf("%s",strerror(errno));
        perror("error in shared memory get statement");
        exit(1);
    }

    shm = shmat(shmid, NULL, 0);                         // attach proc virtual mem range to shm obj, with ptr returned
    if(shm == (char*)-1){
        printf("%s", strerror(errno));
        perror("error in shared memory attachment");
        exit(1);
    }

    for(s = shm; *s != 0; s++){                          // print shm content
        printf("%c", *s);
    }
    printf("\n");
    *shm = '*';                                          // send sig by shm to program 1
    return 0;
}
```

$ 3 compile the program:

```text
$ gcc -o a a.c
$ gcc -o b b.c
```

$ 2 execute program 1 in background, shm obj created, waiting for program 2 to communicate on:

```text
$ ./1 &
[1] 7556
```

$ 3 check mem obj existence by ipcs tool (can only show system-v info):

```text
$(root) ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
0x00000000 2          gdm        777        8192       1          dest
0x00000000 5          gdm        777        1572864    2          dest
0x00000000 10         guolinp    777        16384      1          dest
0x00000000 11         guolinp    777        7520256    2          dest
0x00000000 98318      jiaguoz    600        16777216   2          dest
0x00000000 17         guolinp    600        524288     2          dest
0x00000000 18         guolinp    777        7520256    2          dest
0x00000000 98324      jiaguoz    777        1769472    2          dest
0x00000000 21         guolinp    600        524288     2          dest
0x00000000 22         guolinp    600        524288     2          dest
0x00000000 163863     jiaguoz    777        3440640    2          dest
0x00000000 27         guolinp    600        16777216   2          dest
0x00000000 32803      jiaguoz    777        16384      1          dest
0x00000000 32807      jiaguoz    777        6651904    2          dest
0x00000000 32810      jiaguoz    600        524288     2          dest
0x00000000 32811      jiaguoz    777        6651904    2          dest
0x00000000 32814      jiaguoz    600        524288     2          dest
0x00000000 32815      jiaguoz    600        524288     2          dest
0x00000000 163893     jiaguoz    777        2801664    2          dest
0x00002694 163896     metung     666        100        1                   <---- the shm obj existed
0x00000000 58         guolinp    600        524288     2          dest
0x00000000 59         guolinp    777        7077888    2          dest
0x00000000 65599      jiaguoz    600        524288     2          dest
```

$ 4 execute program 2, signal to program 1 for cleanups.

```text
$ ./2
hello world
[1]-  Done                    ./1                                 // program 1 clear up the shm obj & exit

$ ./2                                                             // execute program 2 again, no shm obj existed
Error in Shared Memory get statement: No such file or directory
```

$ 5 check mem obj existence by ipcs tool again:

```text
$(root) ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
0x00000000 2          gdm        777        8192       1          dest
0x00000000 5          gdm        777        1572864    2          dest
0x00000000 10         guolinp    777        16384      1          dest
0x00000000 11         guolinp    777        7520256    2          dest
0x00000000 98318      jiaguoz    600        16777216   2          dest
0x00000000 17         guolinp    600        524288     2          dest
0x00000000 18         guolinp    777        7520256    2          dest
0x00000000 98324      jiaguoz    777        1769472    2          dest
0x00000000 21         guolinp    600        524288     2          dest
0x00000000 22         guolinp    600        524288     2          dest
0x00000000 163863     jiaguoz    777        3440640    2          dest
0x00000000 27         guolinp    600        16777216   2          dest
0x00000000 32803      jiaguoz    777        16384      1          dest
0x00000000 32807      jiaguoz    777        6651904    2          dest
0x00000000 32810      jiaguoz    600        524288     2          dest
0x00000000 32811      jiaguoz    777        6651904    2          dest
0x00000000 32814      jiaguoz    600        524288     2          dest
0x00000000 32815      jiaguoz    600        524288     2          dest
0x00000000 163893     jiaguoz    777        2801664    2          dest
0x00000000 58         guolinp    600        524288     2          dest
0x00000000 59         guolinp    777        7077888    2          dest
0x00000000 65599      jiaguoz    600        524288     2          dest
```

no shm obj existed anymore.
