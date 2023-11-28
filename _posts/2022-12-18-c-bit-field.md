---
layout: post
title: "bit field (c)"
author: "twistfatezz"
date: 2022-12-18 15:19
categories: "2022"
tags:
  - c
---
### # definition 
The storage of some data doesn't need to take up a whole byte, merely a few bits is enough. For example, the switch variable only has 2 state: on and off. C language provide a certain struct/union called bit field to enable this feature.
```c
struct bitfield{
    unsigned int m; // take up 4 bytes = 32 bits
    unsigned int n:4; // take up 4 bits 
    unsigned char ch:6; // take up 6 bits
}
```

<hr>

### # truncation Effect of bit field
The bit field restricted the number of bits to use, the excess part will be truncated.
```c
#include <stdio.h>

typedef struct BitFields{
    unsigned int m; // <= 32 bits
    unsigned int n:4; // <=4 bits
    unsigned char ch:6; // <= 6 bits
} bitfield;

int main(){
    bitfield bf1 = {0xad, 0xE, '#'};
    printf("0xad, 0xE, '#'  =>  %#x, %#x, %c\n", bf1.m, bf1.n, bf1.ch);
    bitfield bf2 = {0xadadad, 0xEE, '$'};
    printf("0xadadad, 0xEE, '$'  =>  %#x, %#x, %c\n", bf2.m, bf2.n, bf2.ch);
    return 0;
}
```
```txt
0xad, 0xE, '#'  =>  0xad, 0xe, #
0xadadad, 0xEE, 'z'  =>  0xadadad, 0xe, :
```
To explain why char `z` is converted to `:`, we should take a look at their ASCII code: `0x7a=111 1010` and `0x3a=11 1010`, and found that the latter one is 6-bit truncated of the former one.

<hr>

### # two Limitations
$1 The width of bit field must be less than the length of its attached type.

$2 According to c standard, the following type now supported bit field feature: int, unsigned int, signed int, \_Bool. 

Note: some implementation of compiler may also support other types like: char, signed char, unsigned char, enum, etc. 

<hr>

### # bit field Memory Model
The memory model of bit field is not covered at c standard, thus it differs among compiler implemantations, but to compress the occupied space as much as possible.

$1 When the adjacent bit fields are of the same type, and the sum of their width is less than the length of the type, then the bit field members will piled with each other until exceed the type bits limit. If the sum is above the length of the type, the first intolerable member will offset to next block of type length.
```c
#include <stdio.h>

typedef struct bitfields{
    unsigned int m:6;
    unsigned int n:12; // 6+12<32
    unsigned int p:4; // 6+12+4<32
} bitfield;

typedef struct bitfield2s{
    unsigned int m:22; 
    unsigned int n:12; // 22+12>32
    unsigned int p:4; // 22+12+4<32 
} bitfield2;

typedef struct bitfield3s{
    unsigned int m:22;
    unsigned int n:12; // 22+12>32
    unsigned int p:22; // 12+22>32
} bitfield3;

int main(){
    printf("size of bitfield: %lu\n", sizeof(bitfield));
    printf("size of bitfield2: %lu\n", sizeof(bitfield2));
    printf("size of bitfield3: %lu\n", sizeof(bitfield3));
    return 0;
}
```
```txt
size of bitfield: 4
size of bitfield: 8
size of bitfield: 12 
```
the 3 members are the same type, if n+m+p<32, then the 3 member will piled with each other in the same 4 bytes unsigned int space.

$2 When the adjacent members are not the same type, the alignment of members differs from compilers.
```c
#include <stdio.h>

typedef struct bitfields{
    unsigned int m:12;
    unsigned char n:4;
    unsigned int p:4;
} bitfield;

int main(){
    printf("size of bitfield: %lu\n", sizeof(bitfield));
    return 0;
}
```
the different compiler may give different result: if the members are compressed, then the result will be 4, otherwise may get 12.

<hr>

### # appendix
$1 The align of bit field members sometimes not at the beginning of bytes, so it's not reasonable to get the address of the member with bit field. The address is the number of bytes but not bits.

$2 todo

