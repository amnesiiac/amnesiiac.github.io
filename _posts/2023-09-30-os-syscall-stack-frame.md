---
layout: post
title: "syscall: stack frame analysis when trap (operating system)"
author: "melon"
date: 2023-09-30 11:01
categories: "2023"
tags:
  - os
---

### # stack frame changes & context transition
This part mainly describe the stack frame changes & context transition when process trapped from user to kernel mode.

The src code segment below is a simple "helloworld" print program, inside which a user mode to kernel mode context swtich happens:
```text
#include <unistd.h>                            // unistd: POSIX API without i/o buffer mechanism

int main(){
    char msg[] = "hello, world\n";
    write(STDOUT_FILENO, msg, sizeof(msg)-1);  // call POSIX glibc API write
    return 0;
}
```
In this article, we take the above code as a case to describe the flow of trap exception between user/kernel switches.  

<hr>

### # user mode execution flow (before trap exception occurs)
Some preliminary info:  
On x86_64 arch systems, __rbp__ and __rsp__ register are for saving state of process stacks.  
The __rbp__ hold the base stack address of the process stack frame, all data from the function are accessed by __rbp__ + offset.
The __rsp__ hold the top stack address of the process stack frame.

User mode program asm code segment:
```text
push    %rbp                       ; preserve the state before exeucte main function
mov     %rsp,%rbp                  ; setup main function stack base ptr rbp to the location of rsp
sub     $0x10,%rsp                 ; move rsp down by 16 byte (16 byte alignment ABI convention)
movabs  $0x726f776f6c6c6568,%rax   ; move ascii of "hellowor" to rax
mov     %rax,-0x10(%rbp)           ; save content of rax to rbp-16byte
movl    $0xa646c,-0x8(%rbp)        ; move ascii of "ld" to rbp-8byte
lea     -0x10(%rbp),%rax           ; save addr of stack top to rax
mov     $0xb,%rdx                  ; save param3 of "write" to rdx: length of "helloworld\n" (11)
mov     %rax,%rsi                  ; save param2 of "write" to rsi: content to be print out (rax)
mov     $0x1,%rdi                  ; save param1 of "write" to rdi: STDOUT_FILENO (1)
callq   0x400400 <write@plt>       ; callq syscall
```

<hr>

### # kernel mode execution flow (after trap exception occurs)
```text
mov            $0x1,%rax           ; setup syscall write number in rax reg
0x7ffff7afcb9e syscall             ; set ret addr in rip reg, and call syscall to trap in kernel mode
```                           
the flow chart depict the trap from user mode to kernel mode, then ret:
```txt

   program                 cpu regs                        stack frame addr model
  ┌─────────────────────┐ ┌─────────────────────────────┐ ┌─────────────────────┐                 
  │   ┌─────────────┐   │ │rax: 0x7fffffffe520 -> 0x1   │ │                     │ 0x7fffffffe538  
  │   │    write    │   │ │rdx: 0xb            (param3) │ ├─────────────────────┤                 
  │   └──────┬──────┘   │ │rsi: 0x7fffffffe520 (param2) │ │                     │ 0x7fffffffe530 (rbp = prev rsp)
  │ ┌────────+────────┐ │ │rdi: 0x1            (param1) │ ├─────────────────────┤                 
  │ │     libc.so     │ │ │rip: the addr for ret from   │ │ $0xa646c            │ 0x7fffffffe528 (rbp-8byte)
  │ │ ┌─────────────┐ │ │ │     syscall  ^              │ ├─────────────────────┤                 
  │ │ │   syscall   │ │ │ └──────────────┼──────────────┘ │ $0x726f776f6c6c6568 │ 0x7fffffffe520 (rsp = rbp-16byte)
  │ │ └─────────────┘ │ │                |                └─────────────────────┘                 
  │ └─────────────────┘ │                |
  └──────────┬──────────┘                └----------------------------------------------┐
             │                                                                          |
      syscall│                                                                          |                user space
========================================================================================|==========================
          ┌──+──┐          ┌-------->struct pt_regs                                     |              kernel space
  ┌───────┤trap ├───────┐  |        ┌────────────────────────────────────┐              |
  │       └──┬──┘       │  |  ┌----->ax: syscall number -> func ret state│              |
  │ ┌────────+────────┐ │  |  |     │dx: param3                          │              |
  │ │     MSR regs    │ │  |  |     │si: param2                          │              |
  │ └────────┬────────┘ │  |  |     │di: param1                          │              |
  │   entry_SYSCALL_64  │  |  |     │long cx: addr of next instruction   │              |
  │ ┌────────+────────┐ │  |  |     │# state reg for recover             │              |
  │ │     pt_regs     +-┼--┘  |     │long ip: ip reg                     │              |
  │ │ store ctx/param │ │     |     │long flags: CPU state               │              |
  │ └────────+────────┘ │     |     │long sp: user mode stack top        │              |
  │          │          │  ret|     └────────────────────────────────────┘              |
  │ ┌────────+────────┐ │     |      sys_call_table                                     |
  │ │  do_syscall_64  │ │     |     ┌─┬─────┬──────────────┐                            |
  │ │ ┌──────────────┐│ │     |     │2│open │__64_sys_open │                            |
  │ │ │__64_sys_write+┼-┼-----+----->─┼─────┼──────────────┤                            |
  │ │ └──────────────┘│ │           │1│write│__64_sys_write│                            |
  │ └────────+────────┘ │           ├─┼─────┼──────────────┤                            |
  │        output       │           │0│read │__64_sys_read │                            |
  │ ┌────────+────────┐ │           └─┴─────┴──────────────┘                            |
  │ │   "helloworld"  │ │                                                               |
  │ └────────+────────┘ │                                                               |
  │          │          │                                                               |
  │   ┌──────+──────┐   │                                                               |
  │   │   swargs    │   │            ctx switch, return to pos that trap into sys,      |
  │   │ (ctx switch)+---┼---------------------------------------------------------------┘
  │   └─────────────┘   │            which is recorded in rip reg
  └─────────────────────┘

```                       
the asm code of kernel mode execution:
```text                   
entry_SYSCALL_64                     ; syscall entry func, save rax,rdx,rsi,rdi into MSR regs
call do_syscall_64                   ; call sys_func by sys_call number
regs->ax = sys_call_table[nr](regs)  ; save function execution state into ax reg
USERGS_SYSRET64                      ; recover & return to user mode
```

<hr>

### # appendix
cpu regs:  
1) params regs: rdi, rsi, rdx, rcx, r8, r9.  
2) ret regs: rax.  
3) storage regs: rbx, rbp, r12, r13, r14, r15.

sys_call_table: linux-${version}/arch/x64/entry/syscalls/syscall_64.tbl

