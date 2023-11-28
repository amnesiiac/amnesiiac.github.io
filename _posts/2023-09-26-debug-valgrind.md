---
layout: post
title: "valgrind: memory issues debugger (c/c++)"
author: "melon"
date: 2023-09-26 08:18
categories: "2023"
tags:
  - debug
---

### # invalid read detection
```text
#include <stdio.h>
#include <stdlib.h>

int main(void){
    char* p = malloc(1);    // new p
    *p = 'a';

    char c = *p;            // new c
    printf("\n [%c]\n",c);

    free(p);                // free p
    c = *p;                 // invalid read of char: use value of p after free
    return 0;
}
```
compile the program:
```text
$ gcc -Wall invalid_read.c -g -o out
$ valgrind --tool=memcheck --leak-check=full -s ./out
```
```txt
==15713== Memcheck, a memory error detector
==15713== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==15713== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
==15713== Command: ./k
==15713==

[a]
==15713== Invalid read of size 1
==15713==    at 0x400609: main (k.c:12)
==15713==  Address 0x5205040 is 0 bytes inside a block of size 1 free'd
==15713==    at 0x4C2B06D: free (vg_replace_malloc.c:540)
==15713==    by 0x400604: main (k.c:11)
==15713==  Block was alloc'd at
==15713==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==15713==    by 0x4005CE: main (k.c:5)
==15713==
==15713==
==15713== HEAP SUMMARY:
==15713==     in use at exit: 0 bytes in 0 blocks
==15713==   total heap usage: 1 allocs, 1 frees, 1 bytes allocated
==15713==
==15713== All heap blocks were freed -- no leaks are possible
==15713==
==15713== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
==15713==
==15713== 1 errors in context 1 of 1:
==15713== Invalid read of size 1
==15713==    at 0x400609: main (k.c:12)
==15713==  Address 0x5205040 is 0 bytes inside a block of size 1 free'd
==15713==    at 0x4C2B06D: free (vg_replace_malloc.c:540)
==15713==    by 0x400604: main (k.c:11)
==15713==  Block was alloc'd at
==15713==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==15713==    by 0x4005CE: main (k.c:5)
==15713==
==15713== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

<hr>

### # invalid write detection
```text
#include<stdlib.h>

void k(void){                          // memory leak
    int* x = malloc(8 * sizeof(int));
    x[9] = 0;                          // invalid write of int: arr subscript overflow
    return;
}

int main(void){
    k();
    return 0;
}
```
compile & debug the program:
```text
$ gcc -Wall invalid_write.c -g -o out
$ valgrind --tool=memcheck --leak-check=full ./out
```
```txt
==5804== Memcheck, a memory error detector
==5804== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==5804== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
==5804== Command: ./t
==5804==
==5804== Invalid write of size 4
==5804==    at 0x40054B: k (t.c:6)
==5804==    by 0x40055B: main (t.c:11)
==5804==  Address 0x5205064 is 4 bytes after a block of size 32 alloc'd
==5804==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==5804==    by 0x40053E: k (t.c:5)
==5804==    by 0x40055B: main (t.c:11)
==5804==
==5804==
==5804== HEAP SUMMARY:
==5804==     in use at exit: 32 bytes in 1 blocks
==5804==   total heap usage: 1 allocs, 0 frees, 32 bytes allocated
==5804==
==5804== 32 bytes in 1 blocks are definitely lost in loss record 1 of 1
==5804==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==5804==    by 0x40053E: k (t.c:5)
==5804==    by 0x40055B: main (t.c:11)
==5804==
==5804== LEAK SUMMARY:
==5804==    definitely lost: 32 bytes in 1 blocks
==5804==    indirectly lost: 0 bytes in 0 blocks
==5804==      possibly lost: 0 bytes in 0 blocks
==5804==    still reachable: 0 bytes in 0 blocks
==5804==         suppressed: 0 bytes in 0 blocks
==5804==
==5804== For lists of detected and suppressed errors, rerun with: -s
==5804== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```

<hr>

### # invalid free operation detection: 
```text
#include <stdio.h>
#include <stdlib.h>

int main(void){
    char* p;
    p = (char*) malloc(100);
    if(p){                                       // conditional jump/move depand on uninitialized value
        printf("Memory Allocated at: %s/n",p);
    }
    else{
        printf("Not Enough Memory!/n");
    }
    free(p);
    free(p);                                     // invalid free: repetitively free allocated memory
    free(p);                                     // invalid free: repetitively free allocated memory
    return 0;
} 
```
```text
$ gcc -Wall morefree.c -g -o out
$ valgrind --tool=memcheck --leak-check=full ./out
```
```txt
==4420== Memcheck, a memory error detector
==4420== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==4420== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
==4420== Command: ./out
==4420==
==4420== Conditional jump or move depends on uninitialised value(s)
==4420==    at 0x4E84079: vfprintf (in /usr/lib64/libc-2.17.so)
==4420==    by 0x4E8A4E8: printf (in /usr/lib64/libc-2.17.so)
==4420==    by 0x4005EF: main (morefree.c:8)
==4420==
==4420== Invalid free() / delete / delete[] / realloc()
==4420==    at 0x4C2B06D: free (vg_replace_malloc.c:540)
==4420==    by 0x400618: main (morefree.c:14)
==4420==  Address 0x5205040 is 0 bytes inside a block of size 100 free'd
==4420==    at 0x4C2B06D: free (vg_replace_malloc.c:540)
==4420==    by 0x40060C: main (morefree.c:13)
==4420==  Block was alloc'd at
==4420==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==4420==    by 0x4005CE: main (morefree.c:6)
==4420==
==4420== Invalid free() / delete / delete[] / realloc()
==4420==    at 0x4C2B06D: free (vg_replace_malloc.c:540)
==4420==    by 0x400624: main (morefree.c:15)
==4420==  Address 0x5205040 is 0 bytes inside a block of size 100 free'd
==4420==    at 0x4C2B06D: free (vg_replace_malloc.c:540)
==4420==    by 0x40060C: main (morefree.c:13)
==4420==  Block was alloc'd at
==4420==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==4420==    by 0x4005CE: main (morefree.c:6)
==4420==
Memory Allocated at: /n==4420==
==4420== HEAP SUMMARY:
==4420==     in use at exit: 0 bytes in 0 blocks
==4420==   total heap usage: 1 allocs, 3 frees, 100 bytes allocated
==4420==
==4420== All heap blocks were freed -- no leaks are possible
==4420==
==4420== Use --track-origins=yes to see where uninitialised values come from
==4420== For lists of detected and suppressed errors, rerun with: -s
==4420== ERROR SUMMARY: 3 errors from 3 contexts (suppressed: 0 from 0)
```
```text
$ ./out
```
```txt
*** Error in `./out': double free or corruption (fasttop): 0x00000000012e9010 ***
======= Backtrace: =========
/lib64/libc.so.6(+0x81329)[0x7f0bca50b329]
./out[0x400619]
/lib64/libc.so.6(__libc_start_main+0xf5)[0x7f0bca4ac555]
./out[0x4004f9]
======= Memory map: ========
00400000-00401000 r-xp 00000000 fd:01 1318028                            /home/metung/bin/valgrind/out
00600000-00601000 r--p 00000000 fd:01 1318028                            /home/metung/bin/valgrind/out
00601000-00602000 rw-p 00001000 fd:01 1318028                            /home/metung/bin/valgrind/out
012e9000-0130a000 rw-p 00000000 00:00 0                                  [heap]
7f0bc4000000-7f0bc4021000 rw-p 00000000 00:00 0
7f0bc4021000-7f0bc8000000 ---p 00000000 00:00 0
7f0bca274000-7f0bca289000 r-xp 00000000 fd:01 413572                     /usr/lib64/libgcc_s-4.8.5-20150702.so.1
7f0bca289000-7f0bca488000 ---p 00015000 fd:01 413572                     /usr/lib64/libgcc_s-4.8.5-20150702.so.1
7f0bca488000-7f0bca489000 r--p 00014000 fd:01 413572                     /usr/lib64/libgcc_s-4.8.5-20150702.so.1
7f0bca489000-7f0bca48a000 rw-p 00015000 fd:01 413572                     /usr/lib64/libgcc_s-4.8.5-20150702.so.1
7f0bca48a000-7f0bca64e000 r-xp 00000000 fd:01 396290                     /usr/lib64/libc-2.17.so
7f0bca64e000-7f0bca84d000 ---p 001c4000 fd:01 396290                     /usr/lib64/libc-2.17.so
7f0bca84d000-7f0bca851000 r--p 001c3000 fd:01 396290                     /usr/lib64/libc-2.17.so
7f0bca851000-7f0bca853000 rw-p 001c7000 fd:01 396290                     /usr/lib64/libc-2.17.so
7f0bca853000-7f0bca858000 rw-p 00000000 00:00 0
7f0bca858000-7f0bca87a000 r-xp 00000000 fd:01 396687                     /usr/lib64/ld-2.17.so
7f0bcaa54000-7f0bcaa57000 rw-p 00000000 00:00 0
7f0bcaa76000-7f0bcaa79000 rw-p 00000000 00:00 0
7f0bcaa79000-7f0bcaa7a000 r--p 00021000 fd:01 396687                     /usr/lib64/ld-2.17.so
7f0bcaa7a000-7f0bcaa7b000 rw-p 00022000 fd:01 396687                     /usr/lib64/ld-2.17.so
7f0bcaa7b000-7f0bcaa7c000 rw-p 00000000 00:00 0
7ffe1751a000-7ffe1753b000 rw-p 00000000 00:00 0                          [stack]
7ffe175cf000-7ffe175d2000 r--p 00000000 00:00 0                          [vvar]
7ffe175d2000-7ffe175d3000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
Memory Allocated at: /nAborted
```

<hr>

### # memory leak detection
```text
#include <stdio.h>
#include <stdlib.h>

int main(void){
    int* p = malloc(1);  // new p
    *p = 'x';            // invalid write char (byte 1) to int (byte 4)
    char c = *p;
    printf("%c\n",c);
    return 0;            // memory leak: definitely lost due to p no free
}
```
```text
$ gcc -Wall memleak.c -g -O out
$ valgrind --tool=memcheck --leak-check=full ./out
```
```txt
==2863== Memcheck, a memory error detector
==2863== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==2863== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
==2863== Command: ./out
==2863==
==2863== Invalid write of size 4
==2863==    at 0x400597: main (memleak.c:6)
==2863==  Address 0x5205040 is 0 bytes inside a block of size 1 alloc'd
==2863==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==2863==    by 0x40058E: main (memleak.c:5)
==2863==
x
==2863==
==2863== HEAP SUMMARY:
==2863==     in use at exit: 1 bytes in 1 blocks
==2863==   total heap usage: 1 allocs, 0 frees, 1 bytes allocated
==2863==
==2863== 1 bytes in 1 blocks are definitely lost in loss record 1 of 1
==2863==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==2863==    by 0x40058E: main (memleak.c:5)
==2863==
==2863== LEAK SUMMARY:
==2863==    definitely lost: 1 bytes in 1 blocks
==2863==    indirectly lost: 0 bytes in 0 blocks
==2863==      possibly lost: 0 bytes in 0 blocks
==2863==    still reachable: 0 bytes in 0 blocks
==2863==         suppressed: 0 bytes in 0 blocks
==2863==
==2863== For lists of detected and suppressed errors, rerun with: -s
==2863== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```
