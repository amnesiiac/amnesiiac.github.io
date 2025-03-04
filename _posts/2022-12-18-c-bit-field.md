---
layout: post
title: "bit field (c)"
author: "melon"
date: 2022-12-18 15:19
categories: "2022"
tags:
  - c
---

this article focus on the bit-field.

why we need this feature? a) reduces memory consumption; b) to make our program more efficient and flexible;
c) easy to implement.

sometimes, the storage of some data doesn't need to take up a whole byte, merely a few bits is enough.
for example, the switch variable has 2 state only: on and off.
c language provide a certain struct/union called bit field to enable this feature.

the basic syntax of bit-field is as:

```text
struct {
    data_type member_name : width_of_bit-field;
};
```

data_type: an integer type for determination of the bit-field to be interpreted as;
member_name: the name of the bit-field; width_of_bit_field must be less than the size of specified type.

<hr>

### # truncation effect of bit field
the bit field restrict the number of bits to use, the extra part will be truncated.

```text
#include <stdio.h>

typedef struct BitFields{
    unsigned int m;           // take up 4 bytes = 32 bits
    unsigned int n:4;         // take up 4 bits 
    unsigned char ch:6;       // take up 6 bits
} bitfield;

int main(){
    bitfield bf1 = {0xad, 0xE, '#'};
    printf("0xad, 0xE, '#'  =>  %#x, %#x, %c\n", bf1.m, bf1.n, bf1.ch);
    bitfield bf2 = {0xadadad, 0xEE, '$'};
    printf("0xadadad, 0xEE, '$'  =>  %#x, %#x, %c\n", bf2.m, bf2.n, bf2.ch);
    return 0;
}
```

compile & execute the above program:

```text
$ gcc -o out t.c && ./out
0xad, 0xE, '#'  =>  0xad, 0xe, #
0xadadad, 0xEE, 'z'  =>  0xadadad, 0xe, :
```

to explain why char `z` is converted to `:`, just take a look at their ascii format:
`0x7a=111 1010` and `0x3a=11 1010`, the latter is just 6-bit truncated from the former one.

<hr>

### # bit-field usage limitations
1 the width of bit field assigned must be less than the length of its attached type.  
2 according to c standard, the type supported bit field feature are: int, unsigned int, signed int, \_bool.

note: some implementation of compiler may also support other types like: char, signed char, unsigned char,
enum, etc.

<hr>

### # bit field memory model
the memory model of bit field is not covered by c standard, indeed it differs among compiler implemantations,
but to compress the occupied space as much as possible.

1 when the adjacent bit fields are of the same type, and the sum of their width < the length of the type,
then the bit field members will piled with each other until exceed the type bits limit;
if the sum > the length of the type, the first intolerable member will offset to next block of type length.

```text
#include <stdio.h>

typedef struct bitfields{      // unsigned int = 32
    unsigned int m:6;          // 6 < 32:             4
    unsigned int n:12;         // 6+12 < 32:          0
    unsigned int p:4;          // 6+12+4 < 32:        0
} bitfield;

typedef struct bitfield2s{
    unsigned int m:22;         // 22 < 32:      4
    unsigned int n:12;         // 22+12 > 32:   4
    unsigned int p:4;          // 22+12+4 < 32: 0
} bitfield2;

typedef struct bitfield3s{
    unsigned int m:22;         // 22 < 32:    4
    unsigned int n:12;         // 22+12 > 32: 4
    unsigned int p:22;         // 12+22 > 32: 4
} bitfield3;

int main(){
    printf("size of bitfield: %lu\n", sizeof(bitfield));
    printf("size of bitfield2: %lu\n", sizeof(bitfield2));
    printf("size of bitfield3: %lu\n", sizeof(bitfield3));
    return 0;
}
```

compile & execute the above program:

```text
size of bitfield: 4
size of bitfield: 8
size of bitfield: 12 
```

<p style="margin-bottom: 20px;"></p>

2 if the adjacent members are not the same type, the alignment of members differs from compilers.

```text
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

the different compiler may give different result: if the members are compressed, then the result will be 4,
otherwise may get 12.

<p style="margin-bottom: 20px;"></p>

3 why can't we get address of a bit field member?  
the align of bit field members sometimes not at the beginning of bytes, so it's not reasonable to get addr of
the member with bit field.
the address is always computed in the byte's world but not bits.

<hr>

### # bit-field code examples
1 c program to illustrate the structure without bit field:

```text
#include <stdio.h>

struct date {
    unsigned int d;     // 4
    unsigned int m;     // 4
    unsigned int y;     // 4
};

int main(){
    printf("size of date is %lu bytes\n", sizeof(struct date));
    struct date dt = {31, 12, 2014};
    printf("date is %d/%d/%d", dt.d, dt.m, dt.y);
}
```

```text
$ gcc -o out t.c && ./out
size of date is 12 bytes
date is 31/12/2014
```

<p style="margin-bottom: 20px;"></p>

2 c program to demonstrate use of bit-fields usage for minimize mem footprint:

```text
#include <stdio.h>

struct date {
    int d : 5;          // d has value between 0 and 31, so 5 bits are sufficient
    int m : 4;          // m has value between 0 and 15, so 4 bits are sufficient
    int y;
};

int main(){
    printf("size of date is %lu bytes\n", sizeof(struct date));
    struct date dt = {31, 12, 2014};
    printf("date is %d/%d/%d", dt.d, dt.m, dt.y);
    return 0;
}
```

```text
$ gcc -o out t.c && ./out
size of date is 8 bytes
date is -1/-4/2014
```

why the output date out is shown as negative numbers?  
the day value 31 stored in 5 bit signed integer which is equal to 11111 (msb is a 1),
so it’s a negative number (interpreted as signed int) and you need to calculate the 2' complement code
of the binary number to get its actual value: the complement code is 00001, which is equivalent to
the decimal number 1 and finally result in a negative number -1.
similarly, the month 12 in 4-bit representation as 1100, after calculating 2’s complement we get -4.

<p style="margin-bottom: 20px;"></p>

3 c program to illustrate the use of forced alignment in bit-field.

```text
#include <stdio.h>

struct test1 {               // a structure without forced alignment
    unsigned int x : 5;      // 5 < 32:   4
    unsigned int y : 8;      // 5+8 < 32: 0
};

struct test2 {               // a structure with forced alignment for next boundary
    unsigned int x : 5;      // 5 < 32:   4
    unsigned int : 0;        // *         4  ---> force alignment
    unsigned int y : 8;      // 5+8 < 32: 0
};

int main(){
    printf("size of test1 is %lu bytes\n", sizeof(struct test1));
    printf("size of test2 is %lu bytes\n", sizeof(struct test2));
    return 0;
}
```

```text
size of test1 is 4 bytes
size of test2 is 8 bytes
```

<p style="margin-bottom: 20px;"></p>

4 c program to demonstrate the pointers cannot point to bit field members directly: forbidden! 

```text
#include <stdio.h>

struct test {
    unsigned int x : 5;
    unsigned int y : 5;
    unsigned int z;
};

int main(){
    struct test t;
    printf("address of t.x is %p", &t.x);    // uncomment the following line will make the program compile and run
    printf("address of t.z is %p", &t.z);    // the below line works fine as z is not a bit field member
    return 0;
}
```

```text
$ gcc -o out t.c && ./out
t.c: In function ‘main’:
t.c:11:5: error: cannot take address of bit-field ‘x’
```

<p style="margin-bottom: 20px;"></p>

5 c program to show what happends when out of range value is assigned to bit field member? truncated!

```text
#include <stdio.h>

struct test {
    unsigned int x : 2;
    unsigned int y : 2;
    unsigned int z : 2;
};

int main(){
    struct test t;
    t.x = 5;                  // assign the value 5 (110) to x (2 bits)
    printf("%d", t.x);        // print the value of x --> truncated
    return 0;
}
```

```text
$ gcc -o out t.c && ./out
t.c: In function ‘main’:
t.c:11:5: warning: large integer implicitly truncated to unsigned type [-Woverflow]
     t.x = 5;                  // assign the value 5 to x (2 bits)
     ^
1
```

<p style="margin-bottom: 20px;"></p>

6 c program to illustrate that it's not allowed to have array bit field members.

```text
#include <stdio.h>

struct test {
    unsigned int x[10] : 5;   // structure with array bit field
};

int main(){
    return 0;
}
```

```text
$ gcc -o out t.c && ./out
t.c:4:5: error: bit-field ‘x’ has invalid type
     unsigned int x[10] : 5;   // structure with array bit field
     ^
```
