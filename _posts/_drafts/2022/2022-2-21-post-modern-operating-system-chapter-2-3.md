---
layout: post
title: "modern operating system: 进程&线程(3)"
subtitle: '[modern operating system] - chapter2 - interprocess communication' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2022-02-15 10:33
lang: ch 
catalog: true 
categories: operating-system 
tags:
  - Time 2022
  - modern operating system 
---
## 本章概述(3) - 进程间通信
操作系统中最核心的概念就是进程(process)，它是操作系统对于正在运行的程序的一个抽象；操作系统的其他所有内容都是围绕进程概念展开的。

进程是操作系统提供最早的、最重要的抽象概念。即使物理上可用的CPU只有一个，借助这种抽象，也可以虚拟出多个virtual CPU，从而使计算机支持(伪)并发操作的能力。

本文将介绍进程、线程的概念。

### 三：进程间通信(interprocess communication, IPC)
进程通常需要与其他进程通信。例如，在shell pipeline中，第一个进程的输出必须传送给第二个进程，这个传输过程需要进程间通信，且最好使用"具有良好结构(well structured)"的方式，不要使用中断方式。

本文将主要围绕如下3个问题进行展开：<br>
➀ 进程如何将信息传递给其他进程，IPC机制如何实现。<br>
➁ 确保多个进程不会在关键事件上产生"分歧"，即进程之间的争夺不应影响完成事件的效率。例如，在飞机订票系统中，为不同客户处理订单的多个进程为飞机最后的座位的争夺将导致飞机订票系统延时。<br>
➂ 多个进程需要保证按照正确的顺序完成各自的任务。例如，进程A负责产生数据，进程B负责打印数据，那么在B执行打印之前必须等待A进程完成(pthread_join)。

关于上述3个问题，对于线程而言同样适用。
➀ 线程如何将信息传递给其他线程。同一进程下的线程，共享地址空间，线程间信息传递较为容易；不同进程下的线程之间的通信，同进程间通信类似。<br>
➁/➂ 同样适用于线程，并且这两个问题线程同进程一样，可以用同样的方法解决。

##### 3.1 进程的竞争性状态(race conditions)
操作系统中，相互协作的进程可能共享一些公用存储区，这种公用存储区可以位于内存中(e.g. 内核数据结构中)，也可以是硬盘中共享文件；公共存储区的位置不影响IPC的本质以及相关的问题的研究。

▶︎ __结合案例理解IPC如何工作 - 以脱机打印程序为例__ <br>
脱机打印程序(print spooler)中的进程想要打印一个文件时，它将文件名放在一个特殊的脱机目录下(spooler directory)；打印机守护进程则周期性的检查是否有文件需要打印，若存在待打印文件，则打印该文件并从spooler dir中删除该文件名。

➤ 两个进程同时读写共享内存空间 - 结果取决于进程的精确时序<br>
脱机目录下有许多槽位(slot)，编号分别为0、1、2...，每个槽位用于存放一个文件名。假设有两个进程共享变量：out(指向下一个待打印文件)，in(指向dir中下一个空闲的槽位)；将这两个变量保存在所有进程都能访问的文件中，该文件长度为2word。
<center><img src="/img/in-post/operating_system_img/os_2_23.pdf" width="80%"></center>
如上图所示，某时刻，脱机目录中0-3号槽位为空(相应文件已经打印完毕)，4-6号槽位被占用(待打印文件名队列)；进程A和进程B同时想将文件写入7号槽位以加入打印队列。

考虑如下情况：进程A读取共享in变量值为7，将其存在进程内局部变量next_free_slot中。此时发生时钟中断，CPU认为进程A已经运行了足够的时间(事实上没有)，于是进程切换到B，B读取共享in变量值为7，将其存在进程内局部变量next_free_slot中。此时，进程A/B都认为脱机目录中下一个可用槽位是7。

B进程继续执行，将文件名1存储在next_free_slot=7号槽位，并更新共享变量in=8，进程B主动放弃CPU，暂时执行其他工作。A进程继续执行，将文件名2存储在next_free_slot=7号槽位(覆盖掉文件名2)，并更新共享变量in=next_free_slot+1=8。这样，打印守护进程并不能发现错误，进程B得不到任何打印输出。

2个或以上的进程读写共享数据时，最后结果取决于它们运行的精确时序，称为竞争性状态。

##### 3.2 进程的临界区(critical regions) - 避免进程进入竞争性状态
要避免进程进入竞争性状态，关键是要阻止多个进程同时读写共享的数据，即通过程式设计将不同进程对于共享数据的操作变为互斥关系(mutual exclusion)。将对共享内存(数据)进行访问的程序片段称为临界区。为了避免进程进入竞争性状态，本质上是保证任何一个进程在临界区代码执行时的整体原子性。关于原子操作在本文后续部分有简单介绍。

避免进入竞争性状态，保证使用共享数据的并发进程能够正确、高效地执行和协作，需要满足4个条件：<br>
➀ 任何两个进程不能同时处于临界区。<br>
➁ 进程本身不应对CPU执行程序的速度、数量作任何假设；即进程不应对程序执行时序做任何假设。<br>
➂ 临界区外运行的进程不得阻塞其他进程。<br>
➃ 不能使进程无限期地等待以进入临界区。

▶︎ **无视临界区出现多进程竞争，以及利用临界区避免进程进入竞争性状态，2种场景下的时序图** <br>
<center><img src="/img/in-post/operating_system_img/os_2_24.pdf" width="100%"></center>
图1中，由于CPU无视了A进程临界区，在A进程的临界区内执行B进程，导致出现多进程竞争地修改共享数据空间中的变量，因此打印机守护进程无法正常打印B进程。<br>
图2中，展示了利用了多进程临界区后，避免了多进程竞争，从而使得多进程正常修改共享变量，因此打印机守护进程可以正确的打印两个进程的文件。

##### 3.3 忙等待(busy waiting)
单CPU下，当某进程进入临界区后，其他进程因为无法进入竞争性状态(无法和运行态进程构成CPU资源竞争关系)，从而不断探测是否达到竞争性状态边界的过程，称为进程的忙等待。

> ➤ 忙等待概念解析 <br>
> __Busy waiting, also known as spinning(自旋), or busy looping__ is a process synchronization technique in which a process/task waits and constantly checks for a condition to be satisfied ❮before proceeding with its execution❯. <br>
> In busy waiting, a process executes instructions that test for the entry condition to be true, such as the availability of a lock or resource in the computer system.

> ➤ 忙等待应用于实现进程间互斥 & 相关概念解析 <br>
> __Busy looping__ is usually used to achieve ❮mutual exclusion❯ in operating systems. <br>
> __Mutual exclusion__ prevents processes from accessing a shared resource simultaneously. A process is granted exclusive control to resources in its ❮critical section❯ without interferences from other processes in mutual exclusion. <br>
> __A critical section__ is a section of a program code where concurrent access must be avoided.

> ➤ 忙等待机制的缺点 <br>
> __problem__: In some operating systems, busy waiting can be inefficient because the looping procedure is a waste of computer resources. <br>
> __explain__: The system is left idle while waiting which is particularly wasteful if the task/process at hand is of low priority. Resources that can be diverted to complete high-priority tasks are hogged by a low-priority task in busy waiting. <br>
> __solution_1__: The use of a delay function which is implemented in most operating systems can solve the problem. Also known as a sleep system call, a delay function places the process involved in busy waiting into an inactive state for a specified amount of time. In this case, resources are not wasted as the process is "asleep". After the sleep time has elapsed, the process is awakened to continue its execution. If the condition is still not satisfied, the sleep time is incremented until the condition can be satisfied. <br>
> __solution_2__: Another approach is to modify the definition of the waiting procedure to accommodate blocking processes with semaphores(将等待状态分解成阻塞、就绪、运行). A process in busy waiting is blocked and placed on a waiting queue where it does not consume resources. Once the conditions are satisfied, the process is restarted and placed on a ready queue.

> ➤ 忙等待机制的2种适用场景 & 自旋锁简介 <br>
> __benefits_1__: Although inefficient, busy waiting can be beneficial in mutual exclusion if the waiting time is short and insignificant. Additionally, busy waiting is quick and simple to understand and implement. <br>
> __benefits_2__: In some operating systems, busy waiting is beneficial for implementing spinlocks. <br>
> __A spinlock enforces__ a spin/waiting loop on a process that is trying to access a shared resource. i.e it enforces mutual exclusion. Once a spinlock is released, the process continues its execution process. Spinlocks are generally used in operating systems where the number of shared resources is not high enough to cause contentions.

##### 3.4 忙等待概念用于"互斥"进程的实现(mutual exclusion with busy waiting)
本小节讨论几种实现互斥的方案，这些方案都保证：当一个进程在临界区执行操作以更新共享内存时，其他进程不会进入临界区，也不会带来其他麻烦。
    
▶︎ __1 屏蔽中断(disable interrupts)__ <br>
在单CPU下的系统中，最简单的方式是：使每个进程进入临界区后立即disable所有中断(包括时钟中断)，并在进程退出CPU占用前enable所有中断。
CPU只有发生中断时，才会主动进行进程切换，因此屏蔽中断后CPU不会主动切换到其他进程。于是，一旦某个进程屏蔽了中断后，再检查、修改共享内存(进入临界区)，就不必担心其他进程介入。

然而，将屏蔽中断的权限交给用户线程是不明智的：➀ 如果一个恶意进程屏蔽中断后不再打开中断，则整个系统可能因此终止；➁ 另外，这种方案只对单CPU有效，在多CPU下的系统中，屏蔽中断只对执行"disable"指令的CPU有效，其他CPU则照常运行，并且可以访问共享内存。

另一方面，当内核在更新变量、特别是变量列表时，"disable interrupts for a few instructions"对于内核自身而言通常是很方便的。"disable interrupts"通常是操作系统本身的一种有用的技术，但不适合作为用户进程实现互斥机制的通用方法。

▶︎ __2 给共享变量上锁(lock shared var) & "锁变量(varlock)"__ <br>
给共享变量上一把"锁"，这把"锁"的状态由"所变量"数值来表示。"锁变量"的初始值为0。当一个进程想进入"临界区"前，首先检查"锁变量"数值：如果值为0，则进程更新"锁变量"值为1，然后进入临界区执行程式；如果"锁变量"值为1，则进程等待直到其数值变为0。因此，进程通过"锁变量"的数值来判断当前是否有进程在该临界区运行。

然而，此方案和脱机目录案例存在相同的疏漏：假设进程A检查"锁变量"并发现其值为0，恰好在进程A将其值更新为1之前，另一个进程被调度执行，将该"锁变量"值设置为1；当进程A再次运行时，将该"锁变量"设置为1；于是，同时有两个进程处于该临界区中。

更进一步地，为解决上面的问题考虑修改为共享变量"加锁"流程：当进程想进入"临界区"前，首先检查"锁变量"数值，如果值为0，在修改其数值前再次检查"锁变量"数值...<br>
这种改动虽然可以避免前面发生的问题，但是如果进程B恰好在进程A修改"锁变量"数值前再次检查其数值之后，修改了"锁变量"的数值，则同样还会导致进程A/B进入竞争性状态。在多进程编程中不应对进程的时序进行任何假设，因此这种改动不能被采用。

▶︎ __3 严格轮换法 (strict alternation)__ <br>
<center><img src="/img/in-post/operating_system_img/os_2_25.pdf" width="100%"></center>
上图中连续测试"锁变量turn"的数值，直到某个值出现为止，称为忙等待。用于实现进程忙等待的锁称为自旋锁(spinlock)。这种实现进程互斥的方式浪费CPU时间片(timeslice)，除非等待时间非常短，否则不建议使用。

➤ 自旋锁(spinlock)简析 <br>
spinlock：让没有抢到锁的进程在while循环里进行"compare and swap(CAS)轮询"，浪费CPU的时间片资源，直到前面的进程离开临界区(锁变量对应的内存空间被赋值0)。__这个过程涉及运行程序和计算机硬件，不需要操作系统介入。__ 

➤ 原子操作(atomic operation) <br>
原子操作是指：不需要同步机制(synchronized)的操作，不会被进程、线程调度机制打断的操作，这种操作一旦开始就会一直运行到结束，中间不会有任何context switch。<br>
➀ 对于单CPU下的系统中，单个机器指令能够完成的操作都是原子操作，这是因为中断只能发生在指令之间；另外，单CPU下的多进程场景中多条机器指令若为原子操作，需要借助spinlock来保证多条机器指令执行中不会被中断。<br>
➁ 在多CPU下的系统中，多条机器指令构成原子操作的条件则不仅仅能靠spinlock来保证，此时还需要保证当前进程不受其他CPU上的进程的影响。如果多CPU下的进程同时访问内存，产生冲突时，将破坏单CPU中执行操作的原子性。

➤ Linux kernel中自旋锁的实现 <br>
```cpp
// linux kernel 2.6.23 asm-i386/spinlock.h
// lock->slock<0: cpu is taken up by other process;
// lock->slock>=0: cpu in idle state
// LOCK_PREFIX ensures 1 exclusive CPU on memory:
//     早期x86CPU通过<总线锁>来实现LOCK_PREFIX; 现在的x86CPU通过缓存一致性协议实现LOCK_PREFIX
//     缓存一致性: 只有一个cache可以对内存实现最终写入
static inline void __raw_spin_lock(raw_spinlock_t *lock){
    asm volatile("\n1:\t"             // 1:location identifier 
        LOCK_PREFIX " ; decb %0\n\t"  // lock->slock-- 
        "jns 3f\n"                    // if(lock->slock>=0) goto 3: take up the idle cpu
        "2:\t"                        // 2: location identifier
        "rep;nop\n\t"                 // if(lock->slock<0) busy waiting
        "cmpb, %0,%0\n\t"             // check again  =>
        "jle 2b\n\t"                  // if(lock->slock<0) goto 2: busy waiting 
        "jmp 1b\n"                    // if(lock->slock>=0) goto 1: while(true)
        "3:\n\t"                      // 3:location identifier =>
        : "+m" (lock->slock) : : "memory")
}
```

➤ 严格轮换法存在的问题 <br>
当两个进程运行时间相差较大的情况下，使用严格轮换法将导致运行较快的进程的执行被严重拖慢，影响了整体程序执行效率。<br>

➀ 考虑2个处于严格轮换状态的进程，用伪代码展示其互相轮转关系
<center><img src="/img/in-post/operating_system_img/os_2_26.pdf" width="80%"></center>
➁ 在进程#1的执行时间相比进程#0执行时间长很多的情况下，2个进程执行时序图
<center><img src="/img/in-post/operating_system_img/os_2_27.pdf" width="100%"></center>
如上图所示，由于这种情况违反了❮避免竞争性状态4条准则❯中的第3条，因此从T5到T7的时间段内，进程#0一直处于阻塞状态。虽然不会导致进程0/1进入竞争性状态，但是这种阻塞状态导致CPU时间片的浪费。在多个进程的执行时间相差特别大的情况下，这种方案不是很好的备选方案。

▶︎ __4 Perterson Solution (软件方式实现进程临界区互斥)__ <br>
Perterson算法是对Dekker算法的改进精简，通过非常简单的两个函数原型，实现了2进程临界区互斥关系，有效避免了进程陷入竞争性状态。给出的函数原型如下：
```cpp
#define FALSE 0
#define TRUE 1
#define N 2

int turn;
int interested[N];

void enter_region(int process){
    int other; other = 1-process;
    interested[process] = TRUE;
    turn = process;
    while(turn==processs && interested[other]==TRUE);
}
void leave_region(int process){
    interested[process] = FALSE;
}
```
上面的算法程序使用"interested[N]"标识进程是否处于临界区内，"interested"并不能很好的表述算法实际含义，因此使用"hasleft"代替，改写代码如下：
```cpp
#define FALSE 0
#define TRUE 1
#define N 2   // num of processes=2

// shared flags: core of perterson algorithm:
// whether a process can enter the critical region depends on the 2 variables below
int turn;
int hasleft[N];// *

void enter_region(int process){// which process enter?
    int other; other = 1-process;// get another process id 
    hasleft[process] = FALSE;// set not left 
    turn = process;// set flag: current process entering region... 
    // if current process is entering critical region &&
    // other process hasnt left critical region
    while(turn==processs && hasleft[other]==FALSE);// hang up condition
}
void leave_region(int process){// which process leave?
    hasleft[process] = TRUE;// set not interested
}
```

▶︎ __5 TSL instruction (需要硬件支持)__ <br>
有些计算机(尤其是多CPU计算机)具有如下的指令：
```txt
TSL RX,LOCK
```
该指令被称为"测试并加锁"(test and set lock, TSL)，用于将内存字"lock"读取到寄存器RX中，然后在相应地址中存入一个非0值。另外，读字、写字操作保证不可分割，即在TSL指令结束前其他CPU不能访问该内存字，保证了TSL指令的原子性：具体地，执行TSL指令的CPU将总线加锁，以阻塞其他指令在TSL指令结束前访问内存。

➤ 锁住总线≠屏蔽中断 <br>
屏蔽中断只对单CPU下的系统有效，多CPU系统中，其他未设置"disable interrupt"的CPU仍然可以在屏蔽中断的CPU执行TSL read/write操作之间访问内存字。因此，多CPU系统中，唯一能够保证CPU访问内存互斥性的方式就是❮锁住总线❯。给总线加锁需要特殊硬件支持。

➤ TSL指令进入/离开临界区的汇编实现 <br>
```txt
enter_region:          ;标号1 -> busy waiting                                    
    tsl register,lock  ;set register=lock & set lock=1(加锁)
    cmp register,#0    ;判断lock之前的数值是否=0(是否处于解锁态)
    jne enter_region   ;lock之前数值≠0(如果之前处于加锁态) 返回标号1(循环测试) busy waiting...
    ret                ;否则返回调用者 进入临界区
leave_region:          ;标号2
    mov lock,#0        ;设置lock=0(解锁)
    ret                ;返回调用者 离开临界区
```

➤ XCHG指令进入/离开临界区的汇编实现 used by intel x86CPU low-level synchronization <br>
```txt
enter_region:           ;ditto
    mov register,#1     ;set register=1
    xchg register,lock  ;交换register和lock中的内容
    cmp register,#0     ;ditto
    jne enter_region    ;ditto
    ret                 ;ditoo
leave_region:           ;ditoo
    mov lock,#0         ;ditoo
    ret                 ;ditoo
```
➤ TSL/XCHG指令实现互斥算法的缺点 <br>
很容易发现，上述汇编代码算法中，未能成功获取锁的进程/线程将采用"busy waiting"的方式持续检测"加锁条件"，直到能够成功获取锁为止。因此，这种方式浪费了CPU资源，对于进程/线程间时序相差很大的场景下，极大的浪费CPU时间片。<br>
__实际上，由于系统时钟超时作用，进程/线程不会一直处于忙等待状态，系统将调度其他进程/线程执行。__

##### 3.5 睡眠与唤醒(sleep & wakeup)
Perterson算法、TSL、XCHG指令都能保证进程在临界区处于互斥状态，但他们都有忙等待的缺点。本质上，上述3种算法是类似的：进程想进入临界区前，先检查是否允许进入，如果不允许则进程原地等待(busy waiting)，直到允许进入临界区为止。<br>

➤ Peterson算法、TSL、XCHG指令可能出现的问题 <br>
➀ 当进程的执行时间相差较大，则由于进程自旋导致CPU时间片的浪费。<br>
➁ 可能会出现优先级反转问题(priority inversion problem)：<br>
考虑计算机同时运行两个进程H & L，分别具有高优先级/低优先级。给定调度规则：只要高优先级进程处于就绪态(ready)即可运行。某时刻，L进程处于临界区内，H进程切换到就绪态；❮由于H处于就绪态时L进程不会被调度❯，因而无法离开临界区，H进程则在临界区外一直处于忙等待状态。

➤ 使用sleep/wakeup系统调用解决上述问题 <br>
[解决思路]：应该阻塞位于在临界区之外的进程，而不是让他们忙等待以浪费CPU资源。<br>
[解决方案]：借助sleep/wakeup系统调用可以将处于临界区之外的进程挂起，而不是让其自旋。sleep：阻塞调用者直到其他进程将其唤醒；wakeup：唤醒指定进程，接受1个进程标识符作为参数。或者可以使用alternative版本的sleep/wakeup：接受内存地址作为参数，用于匹配sleep/wakeup。

▶︎ __案例分析：生产者/消费者模型__ <br>
两个进程共享同一个固定大小的缓存区。其中一个是生产者(producer)，负责将信息存入缓存区，一个是消费者，从缓存区取出信息；这个模型可以扩展成m个生产者和n个消费者的问题，但这里为了简化，只讨论具有1个生产者/消费者模型。

生产者/消费者模型中可能出现的问题及解决思路：<br>
➀ 当缓存区已满，但生产者仍然想向其中添加一个新数据。solve: 使用"system call:sleep"令生产者休眠，待消费者取出至少1个数据后再调用"system call:wakeup"将其唤醒。<br>
➁ 当婚存取已空，但消费者仍然想从其中取出一个新数据。solve: 使用"system call:sleep"令生产者休眠，待生产者存入至少1个数据后再调用"system call:wakeup"将其唤醒。

▶︎ __sleep/wakeup操作实现producer/consumer模型代码(1)__ <br>
```cpp
#define N 100  // buffer size 
int count = 0;  // num of items in buffer -> shared variable -> critical region
void producer(void){
    int item;
    while(TRUE){
        item = produce_item();// produce item ...
        if(count==N){// if buffer is full
            sleep();// blocked, waiting for consumer's wakeup
        }
        insert_item(item); count++;// ... (shared data ops) 
        if(count==1){// if buffer empty before insert
            wakeup(consumer);// wakeup consumer
        }
    }
}
void consumer(void){
    int item;
    while(TRUE){
        if(count==0){// if buffer is empty
            sleep();// blocked, waiting for producer's wakeup
        }
        item = remove_item(); count--;// ... (shared data ops)
        if(count==N-1){// if buffer full before remove
            wakeup(producer);// wakeup producer
        }
        consume_item(item);// non-critical region operations ...
    }
}
```

▶︎ __代码(1)中存在的问题简析__ <br>
很容易注意到，上面代码中count为共享变量，存放于共享内存中，但是producer/consumer进程代码中并没有保证count相关代码部分(临界区)执行时的原子性。因此，上述代码在执行时可能出现竞争性状态的问题，即2个进程的执行后结果状态由进程的执行时序决定。

例如上述代码可能出现如下的情况：缓存区为空，consumer进程读取count=0，此时调度程序将进程切换到producer，producer向缓存区添加一项数据，count=1，producer发现count=1，因此在添加数据之前count=0(由此推断consumer处于sleep状态)，于是producer将对一个处于唤醒状态的consumer执行"wakeup"操作，wakeup信号丢失。consumer从中断处恢复执行，此时经过"count--"后的下一个while循环中判断count=0，因此consumer进程休眠。此后producer持续向缓存区添加数据直到填满整个缓存区，然后休眠。这样两个进程将永远休眠下去。

▶︎ __模型中竞争性状态问题的解决方案__ <br>
上述代码中出现竞争性状态的直接原因是：producer向处于唤醒状态的consumer的wakeup信号，而该信号的"丢失"将导致producer对consumer产生了错误的判断。很容易想到的一种弥补方法是：采用类似"Peterson算法"中的"hasleft"机制，以记录进程处于唤醒/休眠状态，从而防止wakeup信号"丢失"。

▶︎ __sleep/wakeup操作解决producer/consumer模型竞争性状态问题代码(2)__ <br>
```cpp
#define N 100  // buffer size 
int count = 0;  // num of items in buffer -> shared variable -> critical region
// if a process is awake, set related flag=1
// if a process is asleep, set related flag=0
int wakeup_or_not[2];// 2 processes' related flag

void producer(void){// 0
    int item;
    while(TRUE){
        wakeup_or_not[0]=1;// set init state as awake
        item = produce_item();// produce item ...
        if(count==N){// if buffer is full
            wakeup_or_not[0]=0;// update state
            sleep();// blocked, waiting for consumer's wakeup
        }
        insert_item(item); count++;// ... (shared data ops) 
        if(count==1 && wakeup_or_not[1]==0){// if buffer empty before insert && is asleep
            wakeup(consumer);// wakeup consumer
        }
    }
}
void consumer(void){// 1
    int item;
    while(TRUE){
        wakeup_or_not[1]=1;// set init state as awake
        if(count==0){// if buffer is empty
            wakeup_or_not[1]=0;// update state
            sleep();// blocked, waiting for producer's wakeup
        }
        item = remove_item(); count--;// ... (shared data ops) 
        if(count==N-1 && wakeup_or_not[0]==0){// if buffer full before remove && is asleep
            wakeup(producer);// wakeup producer
        }
        consume_item(item);// non-critical region operations ...
    }
}
```
但是，需要注意的是，即使使用了"wakeup_or_not"用于记录2个进程的状态，但是上述代码不产生竞争性条件的前提是："update state"语句和"sleep"两条指令执行时的原子性，即这两条指令执行过程中不出现中断。<br>
另外，对于多个进程构成的"生产者-消费者模型"，则需要使用更多的状态位用于记录每个进程的"实时"状态，从原则上讲，这种改进方案并没有从根本上解决模型的问题。

##### 3.6 信号量(semaphores)
本小节将介绍一种类似sleep/wakeup算法的方法，用于解决多进程临界区互斥问题。

Dijkstra提出了一种新的变量类型：信号量；该变量可以取0值，表示当前没有"pending wakeup"操作；可以取正整数，表示当前存在1或多个"pending wakeup"操作。Dijkstra还针对信号量提出2种操作：down/up(分别对应上一小节中的sleep/wakeup)。

▶︎ __down/up操作伪代码__ <br>
```cpp
void down(bits){// implement as an <atomic operation>
    if(bits>0){bits--;} 
    if(bits==0){sleep();}// blocked
}
void up(){// implement as an <atomic operation>
    bits++;
}
```

▶︎ __使用down/up操作完成producer/consumer互斥模型建立__ <br>
dijkstra提出的semaphore模型可以解决producer/consumer模型中wakeup信号"丢失"的问题。为了保证semaphore模型正常工作，需要将down/up操作过程中的"不可分割性"，因此，down/up通常实现为"system call:down/up"。
在单CPU系统下，需要为这2个"system call"屏蔽中断；如果该模型用于多CPU系统，则需要考虑使用锁变量保护共享数据semaphore：通过TSL/XCHG指令总线加锁方式实现。

如果每个进程在进入临界区前，执行down操作，并在离开临界区前，执行up操作，则可以保证临界区进程访问的互斥性。

▶︎ __semaphore模型使用3个信号量简析__ <br>
➀ full: 表示被填充的slot的数目，初始值为0。<br>
➁ empty: 表示未被填充的slot的数目，初始值为buffer中slot的数量。<br>
➂ mutex: 二元信号量(binary semaphore)，用于确保producer/consumer进程不会同时访问共享数据区(buffer - critical region)，初始值为1。

▶︎ __down/up操作实现producer/consumer互斥模型代码__ <br>
```cpp
#define N 100
typedef int semaphore;
// shared variables
semaphore mutex = 1;// init binary mutex
semaphore empty = N;// num of empty slots in buffer
semaphore full = 0;// num of filled slots in buffer

void producer(void){
    int item;
    while(TRUE){
        item = produce_item();// produce item ... to put into buf
        down(&empty);// empty_slot-1
        down(&mutex);// entering critical region ->
        insert_item(item);// ... (shared data ops)
        up(&mutex);// leaving critical region ->
        up(&full);// full_slot+1
    }
}
void consumer(void){
    int item;
    while(TRUE){
        down(&full);// full_slot-1
        down(&mutex);// entering critical region ->
        item = remove_item();// ... (shared data ops)
        up(&mutex);// leaving cirtical region ->
        up(&empty);// empty_slot+1
        consume_item(item);// non-critical region operations ...
    }
}
```
需要注意，上述程序中信号量的作用不完全一致：<br>
➀ mutex信号量用于实现进程对临界区访问的互斥，它保证任一时刻只有一个进程读写共享数据部分。<br>
➁ full/empty两个信号量用于实现同步(synchronization)，即用于保证某个进程时序(event sequences)一定发生/不会发生。结合本案例：full/empty信号量用于保证：当buffer填满时producer停止运行，当buffer为空时consumer停止运行。

##### 3.7 互斥量(mutex): 信号量的简化版本
信号量可以保证进程访问临界区的互斥性，同时能够保证producer/consumer不会对buffer执行越界操作；除此而外，信号量可以记录buffer中数据条目数量。如果我们不关心buffer中条目数量，则可以使用互斥量作为替代。

互斥量只适用于管理共享资源/代码；另外，互斥量实现起来简单高效，在实现用户级线程包(package)时非常有用。

▶︎ __互斥量简析__ <br>
互斥量只有2种状态：解锁态(unlocked)、加锁态(locked)。只需一个2进制bit即可表示互斥量，不过实际通常用整型予以表示：0表示解锁态，其他数值表示加锁态。

当进程/线程想访问临界区时，调用互斥锁(mutex lock)，此调用产生2个结果：➀ 如果互斥量处于解锁态(临界区可用)，则调用进程/线程可以访问临界区。➁ 如果互斥量处于加锁态(临界区不可用)，则调用进程/线程阻塞在该互斥量上，直到临界区内的进程/线程执行完毕并调用(mutex unlock)。

特别地，当多个进程/线程几乎同时访问临界区，导致它们都阻塞在一个互斥量上时，在临界区可用的前提下，随机选择一个允许其使用该锁。

▶︎ __mutex_lock/mutex_unlock的汇编实现__ <br>
➤ ➀ TSL版本的实现
```txt
mutex_lock:             ;0
    tsl register,mutex  ;set register=mutex & set mutex=1 
    cmp register,#0     ;判断lock之前的数值是否=0
    jze ok              ;如果lock之前数值=0(之前处于解锁态) 则返回
    call thread_yield   ;lock之前数值≠0(处于加锁态) 则阻塞->调度其他线程执行
    jmp mutex_lock      ;再次尝试获取mutex
ok:                     ;
    ret                 ;返回mutex_lock调用者 进入critical region
mutex_unlock:           ;1
    mov mutex,#0        ;set mutex=0(解锁)
    ret                 ;返回mutex_unlock调用者 离开critical region
```

➤ ➁ XCHG版本的实现
```txt
mutex_lock:
    mov register,#1     ;set register=1
    xchg register,mutex ;swap register & mutex
    cmp register,#0     ;ditto
    jze ok              ;ditto
    call thread_yield   ;ditto
    jmp mutex_lock      ;ditto
ok:                     ;ditto
    ret                 ;ditto
mutex_unlock:           ;ditto
    mov mutex,#0        ;ditto
    ret                 ;ditto
```

➤ 关于互斥量mutex_lock/unlock实现的思考 <br>
很容易注意到互斥量相关mutex_lock/unlock操作同TSL/XCHG指令中的实现类似。<br>
但是，需要注意，TSL/XCHG指令在进程/线程未成功获取锁时，采用"忙等待"方式，浪费CPU资源；而互斥量算法采用阻塞进程方式予以优化。
__实际上，由于系统时钟超时作用，进程/线程不会一直处于忙等待状态，系统将调度其他进程/线程执行。__

##### 3.8 进程间通信算法中的隐含问题、优化方式讨论
▶︎ __部分算法隐含条件：多进程访问共享地址空间的问题__ <br>
对于用户级多线程而言，由于同一进程下的线程共享同一地址空间的特性，多个线程访问同一个互斥量是没有问题的；但是，对于本文前面提到的解决方案(e.g. Peterson算法、信号量法)而言，都有一个隐含的前提：多进程至少应具有访问共享内存的能力。根据chapter2中进程篇介绍的内容可知，多个进程一般具有逻辑上相互独立的地址空间，那么该如何实现算法需要的互斥量共享呢？

如果多个进程可以共享全部、或者大部分地址空间，进程/线程概念的边界将变得模糊；但是，尽管如此，进程/线程之间的差别仍然存在：进程/线程分别管理的资源表仍然不同，进程很难具有用户级线程的执行效率。

▶︎ __快速用户区互斥量futex__ <br>
➤ ➀ 本文前面提及算法在实际应用场景下所面临的问题 <br>
随着并行进程/线程数量的增加，有效的同步、锁机制对提升程序性能而言非常重要。如果进程/线程忙等待时间非常短，则应采用自旋锁方案，否则将极大的浪费CPU时钟周期。如果存在很多资源竞争(contention)，则应采用加锁的方式保护临界区共享数据。<br>
但是，实际情况往往复杂得多，例如：程序开始运行时有很小的竞争，后续进程/线程数增加导致竞争加剧，那么开始就采用阻塞加锁，导致频繁内核切换进程，开销相当大。

➤ ➁ 问题的解决：快速用户空间互斥法(fast user space mutex, futex) <br>
futex是Linux系统中的一个功能特性，它实现了基本的锁机制，而且尽可能避免多进程切换时陷入内核，除非进程切换必须依赖内核实现。内核中的进程切换开销很大，因此futex有助于多进程程序性能提升。

❮futex算法的基本要素、注意事项❯ <br>
futex由两部分组成：内核服务部分(kernel service)、用户库(user lib)。内核服务部分提供了一个"等待队列(wait queue)"，该序列中存放了阻塞在同一个锁上的进程；仅当内核显式启用(unblock)时，才允许这些进程执行。
futex中应当避免创建那些"需要借助开销很大的系统调用(expensive system call)才能加入等待队列的进程"。因此，不存在进程资源竞争时(absense of contention)，futex完全在用户空间中运作。
futex中多进程共享同一个"锁变量"(an aligned 32-bit integer)，该变量被初始化为1(解锁态)。

❮futex算法中锁的获取❯ <br>
线程通过执行原子操作"decrement and test, DAT"来获取锁。在linux系统中，原子操作由定义在头文件中的嵌入在C语言中的汇编asm构成。然后，线程检查DAT操作的结果，以判断锁是否处于释放状态。如果锁处于解锁态，则线程成功获取该锁；如果锁处于加锁态，则当前线程等待。futex lib不会采用自旋，而是通过系统调用将等待状态的线程加入内核中实现的"等待队列"；这种情况下，此类线程切换到内核空间的开销是合理的，因为无论如何该线程都将被阻塞。

❮futex算法中锁的释放❯ <br>
当线程执行完成，它通过原子操作"increment and test, IAT"释放锁变量，并check是否仍然有阻塞在内核"等待队列"中的进程：如果存在，则通知内核将解除1/more个进程的阻塞状态；如果不存在contention，则无需内核空间参与。

▶︎ __pthread中的互斥量/条件变量__ <br>
pthread中提供了许多可以用来同步线程的函数，基本原理是利用一个可以被锁定/解锁的互斥量来保护进程的临界区。

➤ pthread互斥量(mutex) <br>
对于想要访问临界区的线程而言，尝试给相关互斥量加锁(pthread_mutex_lock/trylock)：如果该互斥量尚未加锁，则该线程可立即进入，互斥量自动上锁以阻止其他线程进入；如果该互斥量已经加锁，则调用线程被阻塞直到该互斥量被解锁/返回错误代码。<br>
多个线程在等待同一互斥量解锁，当该互斥量解锁时，处于等待状态的线程中只有一个得以运行(获得该锁)，并将互斥量重新锁定。

➤ pthread中的条件变量(condition variable) <br>
条件变量是pthread提供的除互斥量之外的另一种同步机制(synchronization mechanism)。互斥量在允许/阻塞程序访问临界区时是十分有用的；而条件变量允许因：某些未达条件而阻塞调用线程，例如，即使未能达到释放互斥锁的条件时，也可以由于收到"唤醒信号"的缘由阻塞当前正在运行的调用线程，并释放该锁。

➤ pthread中的条件变量与互斥量一同使用 <br>
条件变量一般和互斥量一同使用，如下面的producer/consumer模型案例所示；此类场景下通常由一个线程锁住一个mutex，然后在该mutex不能达到解锁它的条件时，等待其他线程发来的信号以解锁mutex并阻塞其调用线程(该信号通过condition variable来传递)。

➤ pthread中的条件变量使用注意事项 <br>
条件变量不同于信号量，不会保存在内存中(no memory)，因此，一个信号被发送给某个条件变量，而该条件变量下没有线程处于等待/阻塞状态，则该信号丢失。程序员在设计程序时应当避免信号丢失现象发生。

➤ pthread互斥量/条件变量的主要函数(api) <br>
<center><img src="/img/in-post/operating_system_img/os_2_28.pdf" width="100%"></center>
pthread中的互斥量/条件变量几乎总是同时运用，本文后续部分将介绍线程、互斥量、条件变量之间的交互应用细节。

➤ pthread mutex/condition var机制在producer/consumer模型中的应用 <br>
```cpp
#include <stdio.h>
#include <pthread.h>

#define MAX 1000000000
pthread_mutex_t the_mutex;// shared mutex
pthread_cond_t condc, condp;// condition variable
int buffer = 0;// single slot buffer shared by p/c

void *producer(void *ptr){// asynchronous
    int i;
    for(i=1; i<=MAX; i++){// data(i)
        pthread_mutex_lock(&the_mutex);// lock the mutex or block the calling thread
        while(buffer!=0){// buffer no empty
            pthread_cond_wait(&condp, &the_mutex);// block&wait for a cond(signal) <=
        }
        buffer=i;// producer data ... 
        pthread_cond_signal(&condc);// signal another consumer thread & wake it up =>
        pthread_mutex_unlock(&the_mutex);// release the mutex
    }
    pthread_exit(0);
}
void *consumer(void *ptr){// asynchronous
    int i;
    for(i=1; i<=MAX; i++){// 
        pthread_mutex_lock(&the_mutex);// lock the mutex or block the calling thread
        while(buffer==0){// buffer is empty
            pthread_cond_wait(&condc, &the_mutex);// block&wait for a cond(signal) <=
        }
        buffer=0;// consumer data ...
        pthread_cond_signal(&condp);// signal another producer thread & wake it up =>
        pthread_mutex_unlock(&the_mutex);// release the mutex
    }
    pthread_exit(0);
}
int main(){
    pthread_t pro, con;
    // create mutex
    pthread_mutex_init(&the_mutex, 0);
    // create condition variable
    pthread_cond_init(&condc, 0);
    pthread_cond_init(&condp, 0);
    // see api 
    pthread_create(&con, 0, consumer, 0);
    pthread_create(&pro, 0, producer, 0);
    pthread_join(&pro, 0)
    pthread_join(&con, 0)
    // destroy condition variable
    pthread_cond_destroy(&condc);
    pthread_cond_destroy(&condp);
    // destroy mutex
    pthread_mutex_destroy(&the_mutex);
}
```

##### 3.9 管程(monitor)
▶︎ __管程概念的提出背景：为什么需要管程？__ <br>
借助互斥量、信号量，进程间通信(IPC)的实现似乎变得很容易；但实际上，仍然有许多细节问题需要谨慎，下面结合本文前面"down/up操作实现producer/consumer互斥模型代码"，将producer部分的2个down操作交换次序，分析由于按照错误的顺序更新信号量，导致程序出现死锁的问题。

➤ 案例分析代码 <br>
```cpp
#define N 100
typedef int semaphore;
// shared variables
semaphore mutex = 1;// init binary mutex
semaphore empty = N;// num of empty slots in buffer
semaphore full = 0;// num of filled slots in buffer

void down(bits){// implement as an <atomic operation>
    if(bits>0){bits--;} 
    if(bits==0){sleep();}// blocked
}
void up(){// implement as an <atomic operation>
    bits++;
}
void producer(void){
    int item;
    while(TRUE){
        item = produce_item();// produce item ... to put into buf
        <<<< down(&mutex); >>>>// entering critical region ->
        <<<< down(&empty); >>>> // empty_slot-1
        insert_item(item);// ... (shared data ops)
        up(&mutex);// leaving critical region ->
        up(&full);// full_slot+1
    }
}
void consumer(void){
    int item;
    while(TRUE){
        down(&full);// full_slot-1
        down(&mutex);// entering critical region ->
        item = remove_item();// ... (shared data ops)
        up(&mutex);// leaving cirtical region ->
        up(&empty);// empty_slot+1
        consume_item(item);// non-critical region operations ...
    }
}
```
➤ 上述错误样例代码导致问题分析：<br>
假设当前缓存区处于"填满"状态，此时调用producer模块，由于"down(&mutex)"位于"down(&empty)"之前，因此本应该由后者导致进程/线程阻塞，实际上由前者导致阻塞，此时mutex=0且阻塞调用进程/线程。然后，调用consumer模块，则由于mutex=0的缘故导致consumer进程/线程同样被阻塞。此后，producer/consumer两者将永远持续的阻塞下去，无法正常工作。<br>
__这种运行状态称为死锁(deadlock)，将在chapter6中进行详细讨论。__

即使使用互斥量、信息量实现进程间通信，在编程实现上仍然需要小心谨慎，为了更易于程序的编写，Brinch & Hoare提出了管程(monitor)的概念(primitive)。

▶︎ __管程概念__ <br>
管程是由过程、变量、数据结构等组成的一个集合，如构成一个特殊模块或软件包。管程是语言层面的概念，java支持管程机制实现，而原生c语言并不支持它。<br>
管程的一个重要的特性：任一时刻管程中只能有一个活跃的进程；此特性使得管程实现有效的互斥。

➤ 信号量模型和管程模型的对比分析 <br>
信号量、锁机制都是对计算机硬件底层同步方法的高级抽象方式；而管程则是在信号量机制基础上进行改进的**并发编程模型**。
<center><img src="/img/in-post/operating_system_img/os_2_29.pdf" width="100%"></center>

➤ 管程模型在实现上：<br>
➀ 采用了面向对象的思想封装了线程间的同步控制。<br>
➁ 在任一时刻最多允许一个线程执行管程代码。<br>
➂ 管程允许线程主动释放资源。<br>

➤ 管程模型3种类型 <br>
❮Hansen模型❯ 唤醒线程必须放在代码最后，由此可以保证当前线程唤醒其它线程时已经完成自己的任务。<br>
❮Hoare模型❯ 唤醒线程可以位于代码的任意位置，因此，当唤醒其它线程后，立即阻塞自身，当被唤醒的线程完成操作后，再恢复自己。<br>
❮MESA模型❯ 唤醒线程可以位于代码的任意位置，但被唤醒的线程不会立即执行，而是先加入等待队列等待执行。在这种类型线程模型实现时，需要将条件检查放置在循环中，因为无法保证线程从被唤醒到真正开始执行时唤醒条件是否还能满足。<br>

➤ 管程模型实现producer/consumer模型算法框架 <br>
<center><img src="/img/in-post/operating_system_img/os_2_30.pdf" width="100%"></center>

##### 3.10 消息传递(message passing)
除了基本的producer/consumer模型之外，进程间通信(IPC)的另一种方式是消息传递。消息传递使用send/receive作为源语(primitives)，消息传递的实现类似信息量，但不同于管程：管程是语言层面的实现，而消息传递是通过系统调用来实现；因此消息传递很容易被实现为库例程的一部分。

▶︎ __消息传递系统设计要点__ <br>
➀ 消息传递系统需要确保消息送达目的端，为了防止消息在传递中丢失，需要考虑使用类似tcp/ip协议的方式：接受端对于收到的信息回送ack信号，发送端间隔一定时间后执行消息重发。另外，如果消息被接受端正确接受，但返回的ack丢失，则发送端将重发信息，此时，接受端需要考虑在每条信息中嵌入一个序号来识别可能收到的2条重复消息的时间戳。<br>
➁ 消息系统还需解决进程命名问题，即需要避免send/receive调用中所指定的进程出现二义性。<br>
➂ 消息系统客户端、服务端之间的身份认证也需要考虑，如客户端需要判断它是否和一个真正的文件服务器通信，而不是一个冒充者通信。<br>
➃ 对于消息系统中发送端、接受端处于同一机器上的情况，还需要考虑性能优化上的问题：将消息从一个进程复制到另一个进程通常比信号量操作或送入管程处理要慢。因此，保证系统运转效率也是设计难点之一。

▶︎ __使用消息传递机制解决producer/consumer模型问题__ <br>
考虑使用消息传递机制而不是本文前面提到的共享内存方式来实现producer/consumer问题。

➤ producer/consumer消息模型基本流程 <br>
本案例中使用N条消息，相当于共享内存缓存区中的N个slot；假设所有消息大小相同，并且当发出的消息尚未被接受时，该消息由操作系统自动缓存。具体地：<br>
➀ consumer首先将N条空消息发送给producer。<br>
➁ 当producer向consumer传输一个数据(item)时，它取走一个空消息，并返回填充了内容的消息。<br>
➂ 如果producer创建item的速度比consumer快，则所有空消息将被填满，producer将被阻塞&等待consumer执行以返回新的空消息。<br>
➃ 如果consumer消耗item的速度比consumer快，则所有填满的消息将被空消息代替，consumer将被阻塞&等待producer返回新的填充的消息。

通过上述流程，系统中总消息数保持不变，因此可以将所有消息存储在事先确定大小的内存中。

➤ producer/consumer消息模型中的消息传递方式 <br>
➀ 第一种方式：为每个进程分配唯一的地址，每条消息根据进程地址进行编址：e.g. 类似"段地址+偏移地址"。<br>
➁ 第二种方式：构建一种新数据结构：信箱(mailbox)，用于对一定量的消息进行缓冲。使用信箱时，send/receive调用中的地址参数为信箱地址。

➤ producer/consumer消息模型的实现例程 <br>
```cpp
#define N 100

void producer(void){
    int item;
    message m;// message buffer
    while(TRUE){
        item = produce_item();// produce...
        receive(consumer, &m);// waiting for buffer sent by consumer
        build_message(&m, item);// insert item in message
        send(consumer, &m);// send message to consumer
    }
}
void consumer(void){
    int item, i;
    message m;
    for(i=0; i<N; i++){// send N buffer the first time called
        send(producer, &m);
    }
    while(TRUE){
        receive(producer, &m);// waiting for buffer sent by producer
        item = extract_item(&m);// get item out of buffer
        send(producer, &m);// send empty buffer to producer
        consume_item(item);// consume...
    }
}
```

➤ 关于管程的补充信息 <br>
一个著名的消息传递系统是**消息传递接口(message-passing interface, MPI)**，该系统被广泛应用在科学计算中。

##### 3.11 屏障(barriers)
本文最后一个同步机制(synchronization mechanism)是针对进程组(groups of processes)而不是双进程producer/conusmer场景的。

▶︎ __屏障机制简介__ <br>
一些应用程序被划分成若干**阶段(phases)**，并且规定：仅当所有进程都准备就绪以进入下一个程序阶段时，才允许它们进入下一阶段。该规则借助在应用程序的每个阶段结尾设置屏障来实现：当进程到达屏障后将被阻塞，直到该程序阶段中的所有的进程都到达该屏障。这种程序设计架构保障了进程的同步性。

➤ 屏障机制示意图 <br>
<center><img src="/img/in-post/operating_system_img/os_2_31.pdf" width="100%"></center>
当process A到达应用程序第一阶段结尾(完成相应执行过程)，接着执行barrier库中的原语(primitives)，通常是调用一个库过程(lib procedure)，于是该进程被挂起(suspended)。以此类推，进程B/C同样按照相同机制运行；直到该应用程序第一个阶段中所有进程都到达结尾，所有进程同时被释放。

##### 3.12 避免锁的方法：读-复制-更新(read-copy-update, RCU)
▶︎ __问题提出：多进程/线程并发读写可以不使用锁机制么？__ <br>
"The fastest locks are no locks at all." 那么我们能否允许多进程/线程在不使用锁机制的条件下，对于共享的数据结构进行并行读/写(concurrent read/write)？<br>
显然是不可以的。考虑2个进程的应用程序，进程A对共享空间中的数组进行排序操作，进程B对共享空间中的数据计算均值。当这两个进程并发执行时，进程B的结果随着两个进程之间的调度时序不同而变化，而这肯定是错的。

▶︎ __问题解决思路：引出"read-copy-update"方法__ <br>
然而，在某些情况下，即使其他进程正在使用共享空间中的数据结构，也可以同时通过写操作来更新它。保证这种并发读写能够得到正确的结果的关键是：**确保每个读操作要么读取"旧的"数据版本，要么读取"新的"数据版本，绝不能是"新旧"数据的奇怪混合**。

▶︎ __RCU基本思想__ <br>
首先创建一个"旧的"的共享空间数据的copy，然后writer进/线程更新该copy的内容，最后用copy部分的内容update原始共享空间数据。

▶︎ __RCU流程分析：结合链表数据结构__ <br>
<center><img src="/img/in-post/operating_system_img/os_2_32.pdf" width="100%"></center>
通过上图很容易看出：RCU操作实现链表数据结构并行读写时，在执行操作的任一阶段，reader读取的数据内容要么是publish操作之前的"旧数据"，要么是publish操作之后的"新数据"，不可能是"半新半旧"的脏数据；因此RCU操作可以保证多线程reader/writer操作同一内存空间时，数据的一致性。<br>
另外，为了保证数据一致性，RCU中的操作都实现为原子操作。

▶︎ __RCU流程分析：结合链表数据结构__ <br>
➤ 为树型数据结构添加一个节点 <br>
<center><img src="/img/in-post/operating_system_img/os_2_33.pdf" width="100%"></center>
在RCU执行update(publish)操作之前，所有❮已创建❯的reader线程将引用"旧数据"：即只能调用未添加X节点的树型数据；而在update(publish)操作后，所有❮新创建❯的reader将只能引用"新数据"：即只能调用添加了X节点的树型数据。

➤ 为树型数据结构移除两个节点 <br>
<center><img src="/img/in-post/operating_system_img/os_2_34.pdf" width="80%"></center>
在RCU执行update(publish)操作之前，所有❮已创建❯的reader线程将引用"旧数据"：即只能调用未删除B、C节点的树型数据；而在update(publish)操作后，所有❮新创建❯的reader将只能引用"新数据"：即只能调用删除了B、C节点的树型数据。<br>
需要注意：在(c)中完成update(publish)后，虽然断开了B、C节点和树型数据结构的关联，但是不能马上将它们占用的内存空间回收，而是在(d)中等待所有引用"旧数据"的reader执行完毕后，才进行回收。

▶︎ __RCU流程分析总结：结合上述2个数据结构案例__ <br>
➀ 在RCU执行update(publish)操作前创建的reader线程将只能访问"旧数据"，而在update(publish)操作后创建的reader线程将只能访问"新数据"，RCU操作保证了无论何时创建的reader都不会访问到"半新半旧的脏数据"。<br>
➁ 在对特定数据结构执行某种操作时，如果想要使用RCU方法避免使用锁，则需要遵循：在update(publish)之前保证已创建的reader能够正常访问"旧数据"，此时可以借助相应copy操作将"旧数据状态"进行保存；在update(publish)之后保证后续创建的reader能够正常访问"新数据"。<br>
➂ 需要注意：在执行类似删除、替换等改变原始内存空间数据的操作后，即使按照RCU执行完update(publish)操作后，也不能立即删除对应数据，而是需要等待所有引用了"旧数据"的reader运行完成后，才释放相应存储空间。<br>
➃ RCU能够正确运行的一个重要前提是：update(publish)的整个操作必须为原子操作。<br>

## Reference
> \<modern operating system 4th\> chapter2 <br>
> https://www.baeldung.com/cs/os-busy-waiting <br>
> https://zhuanlan.zhihu.com/p/89439043 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用\`\`将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式：<br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色 <br>
> 11 使用html设置可折叠部分内容：<br>
  `<details>` <br>
      `<summary><b>[点击展开] xxx</b></summary>` <br>
      `<center><img src="/img/in-post/tcp-ip_img/tcp_20_9.pdf" width="100%"></center>` <br>
  `</details>` <br>
> 12 问题脚注: ???problem😫problem???
