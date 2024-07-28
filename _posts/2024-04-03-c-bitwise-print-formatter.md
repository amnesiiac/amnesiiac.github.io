---
layout: post
title: "bit-wise printf formatter (c, utility)"
author: "melon"
date: 2024-04-03 22:01
categories: "2024"
tags:
  - c
---

a simple tricky bit-wise printf formatter with testcase:

```text
#include <stdio.h>
// __int8_t are available both on macos & linux
// #include "stdint.h" --> for int8_t, int16_t...
// #include <cstdint>  --> not found on gcc 4.8.5

#define PATTERN_INT8 "%c%c%c%c%c%c%c%c "
#define BYTE_TO_BINARY_INT8(i)    \
    (((i) & 0x80ll) ? '1' : '0'), \
    (((i) & 0x40ll) ? '1' : '0'), \
    (((i) & 0x20ll) ? '1' : '0'), \
    (((i) & 0x10ll) ? '1' : '0'), \
    (((i) & 0x08ll) ? '1' : '0'), \
    (((i) & 0x04ll) ? '1' : '0'), \
    (((i) & 0x02ll) ? '1' : '0'), \
    (((i) & 0x01ll) ? '1' : '0')

#define PATTERN_INT16 PATTERN_INT8 PATTERN_INT8
#define BYTE_TO_BINARY_INT16(i) BYTE_TO_BINARY_INT8((i) >> 8), BYTE_TO_BINARY_INT8(i)

#define PATTERN_INT32 PATTERN_INT16 PATTERN_INT16
#define BYTE_TO_BINARY_INT32(i) BYTE_TO_BINARY_INT16((i) >> 16), BYTE_TO_BINARY_INT16(i)

#define PATTERN_INT64 PATTERN_INT32 PATTERN_INT32
#define BYTE_TO_BINARY_INT64(i) BYTE_TO_BINARY_INT32((i) >> 32), BYTE_TO_BINARY_INT32(i)

void test_bit_op(){
    unsigned char a = 5;
    unsigned char b = 9;
    printf("a = %d, b = %d\n", a, b);

    printf("a & b:\t" PATTERN_INT8 "\n", BYTE_TO_BINARY_INT8(a&b));
    printf("a | b:\t" PATTERN_INT8 "\n", BYTE_TO_BINARY_INT8(a|b));
    printf("a ^ b:\t" PATTERN_INT8 "\n", BYTE_TO_BINARY_INT8(a^b));
    printf("b << 1:\t" PATTERN_INT8 "\n", BYTE_TO_BINARY_INT8(b<<1));
    printf("b >> 1:\t" PATTERN_INT8 "\n", BYTE_TO_BINARY_INT8(b>>1));
    printf("~a:\t" PATTERN_INT8 "\n", BYTE_TO_BINARY_INT8(~a));
}

void test_diff_len_formatter(){
    __int8_t n0 = 1;
    printf("__int8_t n0: " PATTERN_INT8 "\n", BYTE_TO_BINARY_INT8(n0));

    __int16_t n1 = 4096;
    printf("__int16_t n1: " PATTERN_INT16 "\n", BYTE_TO_BINARY_INT16(n1));

    __int32_t n2 = 1;
    printf("__int32_t n2: " PATTERN_INT32 "\n", BYTE_TO_BINARY_INT32(n2));

    __int64_t n3 = 1;
    printf("__int64_t n3: " PATTERN_INT64 "\n", BYTE_TO_BINARY_INT64(n3));

    printf("\n");
}

int main(){
    test_diff_len_formatter();
    test_bit_op();
    return 0;
}
```

compile & execute the program:
```text
$ gcc -o out hello.c && out
__int8_t n0: 00000001
__int16_t n1: 00010000 00000000
__int32_t n2: 00000000 00000000 00000000 00000001
__int64_t n3: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001

a = 5, b = 9
a & b:	00000001
a | b:	00001101
a ^ b:	00001100
b << 1:	00010010
b >> 1:	00000100
~a:	11111010
```

the above program tested on macos with gcc:
```text
$ gcc --version
Configured with: --prefix=/Library/Developer/CommandLineTools/usr --with-gxx-include-dir=
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/4.2.1
Apple clang version 11.0.3 (clang-1103.0.32.62)
Target: x86_64-apple-darwin20.6.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin
```
