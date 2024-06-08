---
layout: post
title: "a cpp stub trick sample analysis (cpp, stub trick, reverse engineering)"
author: "melon"
date: 2024-03-28 21:23
categories: "2024"
tags:
  - cpp
  - code insight
---

a stub function is mainly to replace origin function with self-designated
stub function in runtime.
this article mainly focus on providing a inline hook style stub code for learning.

<hr>

### # a inline hook style stub usecase analysis
stub.h: TLDR, just take x86_64 arch trunk for analysis, the trick of this inline stub
code is to redirect origin function with self-designed stub function by operation jmp variants.

```text
#ifndef __STUB_H__
#define __STUB_H__

#include <unistd.h>
#include <sys/mman.h>
#include <cstddef>
#include <cstring>
#include <map>

// beautify the addr getter for class::member function
#define ADDR(CLASS_NAME,MEMBER_NAME) (&CLASS_NAME::MEMBER_NAME)

// flush cpu instruction cache for range of mem, to ensure the changes to instruction take effect
#define CACHEFLUSH(addr, size) __builtin___clear_cache(addr, addr+size)

//__i386__ _x86_64__ _M_IX86 _M_X64
#define CODESIZE 13U           /* max code */
#define CODESIZE_MIN 5U        /* near jmp size: 5bytes = jmp(0xE9=1) + operand(32bit=4) */
#define CODESIZE_MAX CODESIZE  /* far jmp size: 13bytes = movabs(0x49 0xbb) + operand(64bit=8) + jmpq(0x41 0xff 0xE3) */

// replace the fn as jmp far to the addr of fn_stub
// movabs $0x0102030405060708,%r11
// jmpq   *%r11
#define REPLACE_FAR(t, fn, fn_stub)\
    *fn = 0x49;                                          /* REX prefix to extend jmp to 64bit */\
    *(fn+1) = 0xbb;                                      /* jmp = addr(fn+1) */\
    *(long long*)(fn+2) = (long long)fn_stub;            /* write addr of fn_stub (64bit) */\
    *(fn+10) = 0x41;                                     /* REX postfix to extend jmp to 64bit */\
    *(fn+11) = 0xff;                                     /* addr(fn+11) = 0xff as opcode */\
    *(fn+12) = 0xe3;                                     /* addr(fn+12) = far jmp to add stored in r11 */\
    CACHEFLUSH((char*)fn, CODESIZE);                     /* flush the cpu cache */

// replace the fn as jmp near to the addr of fn_stub
// jmp rel32 offset_32bit
#define REPLACE_NEAR(t, fn, fn_stub)\
    *fn = 0xE9;                                          /* jmp = addr(fn) */\
    *(int*)(fn+1) = (int)(fn_stub - fn - CODESIZE_MIN);  /* operand: offset from addr of fn to fn_stub */\
    CACHEFLUSH((char*)fn, CODESIZE);                     /* flush the cpu cache */

struct func_stub {
    char* fn;                                            // func name
    unsigned char code_buf[CODESIZE];                    // code_buf for stub func
    bool far_jmp;                                        // jmp type
};

class Stub {
private:
    long m_pagesize;                                     // compilor data model: LP64, long=64bit/int=32bit
    std::map<char*, func_stub*> m_result;
public:
    Stub(){
        m_pagesize = sysconf(_SC_PAGE_SIZE);             // get sys page size
        if(m_pagesize < 0){                              // set default if err
            m_pagesize = 4096;
        }
    }

    ~Stub(){
        clear();
    }

    void clear(){
        std::map<char*, func_stub*>::iterator iter;
        struct func_stub* pstub;
        for(iter=m_result.begin(); iter!=m_result.end(); iter++){                // traverse the stub map list
            pstub = iter->second;
            // mark 2 page after fn as rwx, make sure to contain the fn byte code page range
            if(mprotect(pageof(pstub->fn), m_pagesize*2, PROT_READ|PROT_WRITE|PROT_EXEC) == 0){
                if(pstub->far_jmp){                                              // copy fn back to origin place
                    std::memcpy(pstub->fn, pstub->code_buf, CODESIZE_MAX);
                }
                else{
                    std::memcpy(pstub->fn, pstub->code_buf, CODESIZE_MIN);
                }
                CACHEFLUSH(pstub->fn, CODESIZE);                                 // flush the cpu cache
                mprotect(pageof(pstub->fn), m_pagesize*2, PROT_READ|PROT_EXEC);  // reset 2 page after fn as rx
            }
            iter->second = NULL;                                                 // clear stubbed fn entry
            delete pstub;
        }
        return;
    }

    template<typename T, typename S>
    void set(T addr, S addr_stub){
        char* fn;                                                  // get addr of fn & fn_stub
        char* fn_stub;
        fn = addrof(addr);
        fn_stub = addrof(addr_stub);

        struct func_stub* pstub;                                   // create a placeholder as struct
        pstub = new func_stub;
        reset(fn);                                                 // reset fn/fn_stub back to fn (never stub a stub)
        pstub->fn = fn;

        if(distanceof(fn, fn_stub)){                               // copy fn to placeholder for preservation
            pstub->far_jmp = true;
            std::memcpy(pstub->code_buf, fn, CODESIZE_MAX);
        }
        else{
            pstub->far_jmp = false;
            std::memcpy(pstub->code_buf, fn, CODESIZE_MIN);
        }
        // mark 2 page after fn as rwx, make sure to contain the fn byte code page range
        if(mprotect(pageof(pstub->fn), m_pagesize*2, PROT_READ|PROT_WRITE|PROT_EXEC) == -1){
            throw("stub set memory protect to w+r+x faild");
        }
        if(pstub->far_jmp){                                        // replace fn using fn_stub
            REPLACE_FAR(this, fn, fn_stub);
        }
        else{
            REPLACE_NEAR(this, fn, fn_stub);
        }
        if(mprotect(pageof(pstub->fn), m_pagesize*2, PROT_READ | PROT_EXEC) == -1){
            throw("stub set memory protect to r+x failed");        // reset mem protect rx
        }
        m_result.insert(std::pair<char*, func_stub*>(fn, pstub));  // make stubbed fn info persistent
        return;
    }

    template<typename T>
    void reset(T addr){
        char* fn;
        fn = addrof(addr);
        std::map<char*, func_stub*>::iterator iter = m_result.find(fn);
        if(iter == m_result.end()){
            return;
        }
        struct func_stub* pstub;
        pstub = iter->second;
        if(mprotect(pageof(pstub->fn), m_pagesize*2, PROT_READ|PROT_WRITE|PROT_EXEC) == -1){
            throw("stub reset memory protect to w+r+x faild");
        }
        if(pstub->far_jmp){
            std::memcpy(pstub->fn, pstub->code_buf, CODESIZE_MAX);
        }
        else{
            std::memcpy(pstub->fn, pstub->code_buf, CODESIZE_MIN);
        }
        CACHEFLUSH(pstub->fn, CODESIZE);
        if(mprotect(pageof(pstub->fn), m_pagesize*2, PROT_READ|PROT_EXEC) == -1){
            throw("stub reset memory protect to r+x failed");
        }
        m_result.erase(iter);
        delete pstub;
        return;
    }
private:
    char* pageof(char* addr){                                   // get start addr of the page contains fn
        return (char*)((unsigned long)addr & ~(m_pagesize-1));  // mask bits lower than pagesize > page start addr
    }

    template<typename T>
    char* addrof(T addr){                                       // reinterpreted addr as char* from generic ptr T
        union {                                                 // store all member in same mem loc
            T _s;                                               // original func name type
            char* _d;                                           // output func name type
        } ut;
        ut._s = addr;
        return ut._d;
    }

    bool distanceof(char* addr, char* addr_stub){               // far jmp or near jmp?
        std::ptrdiff_t diff = addr_stub>=addr ? addr_stub-addr : addr-addr_stub;
        if((sizeof(addr) > 4) && (((diff>>31) - 1) > 0)){       // judge if 64 bit system && the diff > 32 bit
            return true;                                        // far jmp: jmp to any mem loc available
        }
        return false;                                           // near jmp rel32 for max 32 bits distance jmp
    }
};

#endif
```

test.cpp: usecase function for normal function stub test.
```text
#include <iostream>
#include "stub.h"

using std::cout;

int foo(int a, int b){
    cout << "this is code in foo function\n";
    return 0;
}

int foo_stub(int a, int c){
    cout << "this is code in stub of foo function\n";
    return 0;
}

int main(){
    cout << "call foo before set stub -> ";
    foo(1, 2);
    Stub stub;
    stub.set(foo, foo_stub);
    cout << "call foo after set stub -> ";
    foo(1, 3);
    cout << "call foo after reset stub -> ";
    stub.reset(foo);
    foo(1, 1);
    return 0;
}
```

makefile:
```text
CXX=g++
CFLAGS=-g -Wall -std=c++11 -w
OUT=out

ROOTSRC=$(wildcard *.cpp)
ROOTOBJ=$(ROOTSRC:%.cpp=%.o)

$(OUT):$(ROOTOBJ)
	$(CXX) $(CFLAGS) -o $@ $^
.cpp.o:
	$(CXX) $(CFLAGS) -c $< -o $@

.PHONY:clean all
clean:
	rm -f *.o $(OUT) $(ROOTOBJ)
```

make & execute the program will print:
```text
$ make
g++ -g -Wall -std=c++11 -pthread -w -c test.cpp -o test.o
g++ -g -Wall -std=c++11 -pthread -w -o out test.o

$ ./out
call foo before set stub -> this is code in foo function
call foo after set stub -> this is code in stub of foo function
call foo after reset stub -> this is code in foo function
```
the func is stubed and then reset successfully.

<hr>

### # reference
opcode ref on x86 (jmp jmpf): http://ref.x86asm.net/coder32.html  
cpu reg ref on x86 (r11): https://wiki.osdev.org/CPU_Registers_x86-64  
c++ abi: https://itanium-cxx-abi.github.io/cxx-abi/abi.html
