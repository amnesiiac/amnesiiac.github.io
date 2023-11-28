---
layout: post
title: "int80: context switch between user/kernel mode (operating system)"
author: "melon"
date: 2023-10-04 14:33
categories: "2023"
tags:
  - os
  - ongoing
---

### # why we need interrupt mechanism?
The following diagram shows the workflow of program exxcution from user mode down to kernel mode.  
And shows how the interrupt mechanism helps cpu to switch between different irrelevant processes.
```txt

                                                            ┌─────────────────────────┐
                               cpu exec program1 (set reg)  │for(int i=0; i<=4; i++){ │
                            ┌-------------------------------+    printf(i);           │
                            |                               │}                        │
                            |  if the ctl reg state: busy   ├─────────────────────────┤
                            ├-------------------------------+      application-1      │
                            |  cpu exec program2 (set reg)  ├─────────────────────────┤
                            |  rather than                  │      application-2      │
                            |  polling the state of ctl reg └────────────┬────────────┘
                            |                                            │
                            |                                     syscall│                     user space
        =================================================================================================
                            |                                            │                   kernel space
             cpu reg        |  interrupt happens                         │         
           ┌──────────────┐ |  copy the val from cpu reg  ┌──────────────│──────────────┐
           │ eax: 4       +-┘  to kernel struct pt_regs   │┌─────────────+─────────────┐│ 
           │ eip: 0x97382 +-----------------------------┐ ││ interrupt service routine ││
           │ esp: 0x38721 │                             | │└─────────────┬─────────────┘│
           │ cr1: 0x78214 +-------------------------┐   | │     save cpu reg state      │ 
           └──────────────┘ polling ctl dev state   |   | │      ┌──────────────┐       │
                                 cpu busy waiting   |   └--------+ eax: 4       │       │
             device controler                       |     │      │ eip: 0x97382 │       │
           ┌────────────────────────────────────────+─┐   │      │ esp: 0x38721 │       │
           │      control reg state: busy or ready    │   │      │ cr1: 0x78214 │       │
           │ ┌──────────────┐ ┌───────────┐ ┌────────┐│   │      └───────┬──────┘       │
           │ │intruction reg│ │control reg│ │data reg││   │     ┌────────+────────┐     │
           │ │(EIP) -> out  │ │(CR0)      │ │(DR)    ││   │     │ out 0x7821 eax  │     │
           │ └──────────────┘ └───────────┘ └────────┘│   │     └─────────────────┘     │
           │  0x7820           0x7924        0x7821   │   └─────────────────────────────┘
           └─────────────────────+────────────────────┘   
                                 │                        
                          ┌──────+───────┐                
                          │console/screen│                
                          └──────────────┘                

```                            
What happens if interrupt is no present?  
The i/o operation are much slower then cpu operations.  
The cpu will be always in busy waiting state, polling the control reg state to be ready.  
With interrupt mechanism, the cpu could switch to another process when the i/o device is busy working, which will make cpu more "flexible & faster".

<hr>

### # 32bit x86 system interrupt handling procedure
given a simple "helloworld" program as:
```text
#include <stdio.h>                  // stdio: encapsulation of lower POSIX API with buffer mechanism

int main(){
    char str[] = "hello, world\n";  // str in mem
    printf(str);                    // print str to console (screen)
    return 0;
}
```
the equivalent form of the above is as:
```text
#include <unistd.h>                            // unistd: POSIX API without i/o buffer mechanism

int main(){
    char msg[] = "hello, world\n";
    write(STDOUT_FILENO, msg, sizeof(msg)-1);  // call POSIX glibc API write
    return 0;
}
```
ref: the procedure is much like syscall-context-switch & syscall-stack-frame  
ref: 12:18 in BV1734y1K7uA
```txt
todo
```
