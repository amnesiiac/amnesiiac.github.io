---
layout: post
title: "modern operating system: 进程&线程(1)"
subtitle: '[modern operating system] - chapter2 - process' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2022-02-08 16:16
lang: ch 
catalog: true 
categories: operating-system 
tags:
  - Time 2022
  - modern operating system 
---
## 本章概述(1) - 进程
操作系统中最核心的概念就是进程(process)，它是操作系统对于正在运行的程序的一个抽象；操作系统的其他所有内容都是围绕进程概念展开的。

进程是操作系统提供最早的、最重要的抽象概念。即使物理上可用的CPU只有一个，借助这种抽象，也可以虚拟出多个virtual CPU，从而使计算机支持(伪)并发操作的能力。

本文将介绍进程、线程的概念。

### 一：进程
严格的讲，一个CPU同时只能运行一个进程。在操作系统设计中，CPU从一个进程快速切换到另外一个进程，进程之间的切换只需要非常短的时间，这样就模拟出"并行"错觉。这种场景被称为"伪并行(pseudo parallelism)"。CPU在进程间的快速切换被称为"多道程序设计(multiprogramming)"

当一个操作系统有2个以上的CPU共享同一物理内存时，此时不同的CPU之间称为"真并行(true parallelism)"。

多个并行的activities很难跟踪，因此操作系统设计者提供了一种概念模型(sequential processes)来方便描述这种并行关系。

##### 1.1 进程模型(sequential processes)
从进程模型角度而言，计算机上所有可运行程序、有时包括操作系统，被组织成若干个sequential processes(简称processes)。一个进程表示一个运行程序实例，包括程序计数器(program counter)，程序相关寄存器(registers)，以及相关变量(variables)。

从概念上讲，每个进程都拥有其独立的virtual CPU；实际上，物理上的CPU不断的从进程之间来回切换。但是，为了更好的理解操作系统原理，相比跟踪CPU在进程间切换，将其看成是在"伪并行"模式下的进程集合是一种更简单的理解抽象。

▶︎ multiprogramming模型(a)及其抽象成的4个独立的"伪并行"sequential processes概念模型(b) 
<center><img src="/img/in-post/operating_system_img/os_2_1.pdf" width="100%"></center>
上图(c)中显示该抽象成的概念模型timeline时序图。同一时刻只有一个process处于active状态。

▶︎ 在对进程编程时，绝不能对进程的时序有任何❮想当然❯的假设 <br>
通过一个相关案例进行理解：考虑一个I/O进程，使用流式磁带机恢复备份文件。该I/O进程执行10000次空循环，以等待磁带机达到正常运行所需的速度，然后该进程发出命令读取磁带中第一个备份记录。<br>
如果在该I/O进程执行空循环等待时间中，CPU决定切换到其他进程，则磁带机中的第一个记录通过磁头时，CPU可能仍然没有切回该I/O进程。

上述案例的关键点是：当一个进程有如此严格的执行时间要求时，即一些特定的程序必须在指定的时刻发生时，就必须采取特殊措施以保证它们一定能够按时发生。幸运的是，大多数进程并不受"伪并行"sequential processes模型、及其他进程时序所影响。

▶︎ ❮进程❯和❮程序❯两个概念之间的区别 <br>
通过一个相关比喻理解两个概念之间的区别：一个厨师在厨房制作蛋糕，做蛋糕的食谱是程序(使用适当形式描述的算法)，厨师本身就是CPU，制作蛋糕的各种原料就是输入数据；那么，进程的概念就是：厨师阅读食谱、获取各种原料、以及烘培蛋糕的整个动作的总和。<br>
如果厨师突然接到一个电话，则他将立即终止制作蛋糕进程，转到电话处理进程上(该进程拥有较高优先级)。

上述案例的关键点是：一个进程是计算机某种活动的抽象，包括：程序的输入、输出、及程序本身的状态。单个CPU可以被多个进程共享，CPU通过某种进程调度算法决定何时终止一个进程的运行，转而为另一个进程提供服务。

另外需要注意的是：即使一个程序被同时执行了两遍，它们也属于两个不同的进程，虽然操作系统使它们在内存中共享一份加载的代码，但那也只是一个和进程概念无关的技术性细节。

##### 1.2 进程的创建(process creation)
对于非常简单的操作系统(e.g. 微波炉控制器系统)，在系统启动时，所有需要的进程都已经存在；这些系统的设计不涉及进程的创建、撤销。然而，大多数系统中，都需要进行进程的创建、撤销。

▶︎ 4种引起进程创建的事件(event) <br>
➀ 系统初始化(system initialization)：启动操作系统会创建若干进程，有些是前台进程(foreground processes)，即同用户进行交互并替他们完成内部工作的进程；另一部分是后台进程(background process)，这些进程和用户没有对应关系，但它们各自具有独立的功能，如邮件接收后台进程，只在收到邮件时被自动唤醒。<br>
例如：对于停留在系统后台的用于处理如电子邮件、Web页面、新闻、打印等活动的进程称为守护进程(daemon)。Unix中可以使用ps指令列出正在运行的进程。

➁ 正在运行的进程(running process)执行了创建进程的系统调用(process-creation system call)：当进程处理的任务可以很容易划分成若干❮相关但没有相互作用❯的子任务时，创建新进程将非常有用。例如：在一个有大量数据需要从网络中获取，并在本地进行顺序处理的场景下，创建一个进程从网络读取数据，并将数据暂存在内存中共享缓存区中，方便让第二个进程读取&处理。这种场景在多核CPU上的运行速度会更快。

➂ 用户请求(user request)创建一个新进程：在交互式系统中，用户键入指令、点击图标就可以启动一个进程，并在其中执行所选择的程序。<br>
例如：在unix、windows系统中，用户可以同时打开多个窗口，每个窗口运行一个进程。

➃ 一个批处理作业(batch job)的初始化：这种场景仅在大型机的批处理系统中发生，大型机的用户(通常是远程)向其提交batch job。当大型机的操作系统判定有足够的资源运行新的进程时，就创建一个进程用于处理batch job queue中的下一个任务。

▶︎ unix系统 [system call: fork] 创建new process的流程 <br>
在unix系统中，只有fork系统调用可以创建新进程。首先，system call: fork创建被调用进程(the parent process)的完全相同的拷贝(the child process)：它们有相同的内存镜像(memory image)、相同的环境字符串(environment strings)、相同的打开文件(open files)状态。然后，the child process执行execve或一个相似的系统调用，来改变其对应的内存镜像，相当于创建、运行了一个全新进程。<br>
例如：当unix用户在shell中键入sort指令后，shell进程fork off出the child process，the child process将负责执行sort指令。<br>
unix分2步完成进程的建立的原因是：允许the child process处理其file descriptors，以在fork system call之后、在执行execve前的时间内，完成标准输入(stdin)、标准输出(stdout)、标准错误(stderr)的重定向(redirection)。

➤ [fork off] 概念解析 <br>
To diverge(划分、分岔) into two or more separate paths (idiomatic, intransitive). Synonymous: branch off.

➤ [file descriptors] 概念解析 <br>
In Unix and Unix-like computer operating systems, a file descriptor(FD) is a unique identifier(handle) for a file or other input/output resource, such as a pipe or network socket. <br>
File descriptors are a part of the POSIX API. Each Unix process (except perhaps daemons) should have three standard POSIX file descriptors, corresponding to the three standard streams: stdin(0), stdout(1), stderr(2). <br>

➤ [process对于file descriptors的处理流程] <br>
<center><img src="/img/in-post/operating_system_img/os_2_2.pdf" width="100%"></center>
To perform input or output, the process passes the file descriptor to the kernel through a system call, and the kernel will access the file on behalf of the process. The process does not have direct access to the file or inode tables.

➤ [parent process和child process的关系案例分析] <br>
```cpp
// --------- fork()用法 ---------
#include <stdio.h>  // printf()
#include <unistd.h>  // getpid() getppid()
#include <sys/types.h>  // pid_t
int main(){
    // parent process和child process之间共享全部的代码
    // parent process从main开始执行, child process从fork返回开始执行
    pid_t pid = fork();// fork(): 根据parent process创建child process
    if(pid==-1){
        printf("fork error!\n");
    }
    else{
        // getpid() return the calling function pid
        // getppid() return the parent function pid
        printf("a example of fork, pid=%d\n", getpid());
    }
    return 0;
}
// output
// a example of fork, pid=43440
// a example of fork, pid=43441
```
```cpp
// --------- parent & child process 各自程序副本的fork()返回值不同 ---------
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
int main(){// parent process从main开始执行
    pid_t pid = fork();// child process从fork返回开始执行
    if(pid==-1){
        printf("fork error!\n");
    }
    else if(pid==0){// child process执行时 fork()返回的pid=0
        printf("pid:%d is in child process\n", pid);
    }
    else{// parent process执行时 fork()返回的pid>0
        printf("pid:%d is in parent process\n", pid);
    }
    return 0;
}
// output
// pid:44305 is in parent process   --- parent process
// pid:0 is in child process        --- child process
```
```cpp
// --------- parent & child process 各自拥有一份程序变量 ---------
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
int glob=10;
int main(){// parent process从main开始执行
    int var=100;
    pid_t pid = getpid();// get calling function pid
    // all the printf use \n to flush the std buf, so only parent process has output here
    printf("before fork:\n");
    printf("pid=%d, glob=%d, var=%d\n", pid, glob, var);
    printf("after fork:\n");
    pid = fork();// child process从fork返回开始执行
    if(pid<0){
        printf("fork error!\n");
        exit(-1);
    }
    else if(pid==0){
        glob++;
        var++;
    }
    else{
        sleep(3);// parent process sleep for 3s, so firstly got the child process ouput
    }
    printf("pid=%d, glob=%d, var=%d\n", getpid(), glob, var);
    return 0;
}
// output
// before fork:                  --- parent process
// pid=44538, glob=10, var=100   --- parent process
// after fork:                   --- parent process
// pid=44539, glob=11, var=101   --- child  process
// pid=44538, glob=10, var=100   --- parent process
```
```cpp
// --------- 文件流缓存区位于用户空间(资源), parent & child process各自拥有一份 ---------
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
int main(){// parent process从main开始执行
    pid_t pid; 
    // 打印输出后 standard buffer已经被\n冲洗(打印的数据已经不在buffer中)
    printf("before fork(), flush buffer\n");
    // 打印输出后 standard buffer未被冲洗(打印的数据仍然在buffer中)
    printf("before fork(), no flush buffer:%d\t",getpid());
    pid = fork();// child process从fork返回开始执行
    if(pid==0){
        printf("\nchild, after fork(): pid=%d\n", getpid());
    }
    else{
        printf("\nparent, after fork(): pid=%d\n", getpid());
    }
    return 0;
}
// output -> (child复制了parent的std buf,在printf刷新buf时,遗留内容和新内容被一起输出)
// before fork(), flush buffer           --- parent process
// before fork(), no flush buffer:45529	 --- parent process
// parent, after fork(): pid=45529       --- parent process
// before fork(), no flush buffer:45529	 --- child  process 
// child, after fork(): pid=45530        --- child  process
```
```cpp
// --------- parent & child process对同一文件执行操作 写入数据不会发生交叉覆盖 ---------
// --------- 说明 它们共享文件偏移量 即共享文件表项 ---------
// Ref. https://blog.csdn.net/peng864534630/article/details/77930555
// Ref. https://www.cnblogs.com/QG-whz/p/5469891.html
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <cstdlib>
#include <string>
#include <fcntl.h>  // O_RDWR|O_CREAT
int main(){// parent process从main开始执行
    pid_t pid;
    int file_desc;// parent & child process分别拥有一份FD拷贝
    int i=1; int status;// status用于收集僵尸子进程退出时的状态
    char ch0[] = "advanced";
    char ch1[] = " programming";
    char ch2[] = " in the unix environment";
    file_desc = open("/Users/mac/desktop/test.txt", O_RDWR|O_CREAT, 0644);
    // open(const char *pathname, int flags, mode_t mode) 
    // access mode: O_RDONLY(读), O_WRONLY(写), O_RDWR(读写)
    // open mode: O_APPEND(追加末尾), O_CREAT(文件不存在则创建), 
    //            O_EXCL(如果设置了O_CREAT且文件存在,则强制打开失败,只和O_CREAT一起使用), 
    //            O_TRUNC(文件存在,为普通文件,有写权限,则清空文件), O_NONBLOCK(以非阻塞模式打开文件)
    // mode: access jurisdiction: S_IRUSR(owner有read权限) S_IWUSR(owner有write权限)
    //                            S_IXUSR(owner有exe权限)  S_IRWXU(owner有read,write,exe权限)
    //                            S_IRGRP(group有read权限) S_IWGRP(group有write权限)
    //                            S_IXGRP(group有exe权限)  S_IRWXG(group有read,write,exe权限)
    //                            S_IROTH(all有read权限)   S_IWOTH(all有write权限)
    //                            S_IXOTH(all有exe权限)    S_IRWXO(all有read,write,exe权限)
    // 创建的文件的访问权限计算方式 => newmode = mode^(~umask)  
    if(file_desc==-1){
        printf("open/create file error:%m\n");
        exit(-1);
    }
    write(file_desc, ch0, strlen(ch0));
    pid = fork();// child process从fork返回开始执行
    if(pid==-1){
        printf("error fork\n");
        exit(-1);
    }
    else if(pid==0){// in child process -> write to the file by file_desc
        i=2;
        printf("in child process\n");
        printf("i=%d\n", i);
        if(write(file_desc, ch1, strlen(ch1)) == -1){// 执行写入
            printf("child write error:%m\n");
            exit(-1);
        }
    }
    else{// in parent process -> write to the file by file_desc
        sleep(1);// waiting for child processs goes firstly
        printf("in parent process\n");
        printf("i=%d\n", i);
        if(write(file_desc, ch2, strlen(ch2)) == -1){// 执行写入
            printf("parent write error:%m\n");
            exit(-1);
        }
        wait(&status);
        // pid_t wait(int *status); pid_t wait(nullptr)杀死僵尸进程、不收集进程退出信息
        // 1 进程一旦调用wait 就会立即阻塞自己
        // 2 wait分析当前进程的某个子进程是否已经退出 
        // 3 如果wait找到僵尸子进程 wait将收集这个进程的信息 将其彻底销毁后返回
        // 4 如果wait没找到僵尸子进程 则wait一直阻塞在这里 直到找到一个僵尸子进程为止
    }
    return 0;
}
// output
// in child process    --- child  process
// i=2                 --- child  process
// in parent process   --- parent process
// i=1                 --- parent process
// shell中执行"cat test.txt"  输出"advanced programming in the unix environment"
```
<center><img src="/img/in-post/operating_system_img/os_2_3.pdf" width="100%"></center>
parent process和child process通过不同的file_desc，共享同一个文件表项，因此两个进程共享同一个文件偏移值，这是它们分别执行write操作写入的数据之间不产生覆盖的原因。

##### 1.3 进程的终止(process termination)
进程的发生终止时通常由以下条件引起：
- ➀ 正常退出(normal exit, voluntary)
    - ➀ 从main函数返回
    - ➁ 调用exit函数
    - ➂ 调用\_exit或\_Exit
    - ➃ 进程的最后一个线程从其启动例程返回(last thread termination) - unix chapter 11.5
    - ➄ 进程的最后一个线程调用pthread_exit(last thread termination) - unix chapter 11.5
    - ➅ 进程的最后一个线程对取消请求作出响应 - unix chapter 12.7
- ➁ 出错退出(error exit, voluntary)
    - ➀ 调用abort函数，abort函数将SIGABRT信号发送给调用进程 - unix chapter 10.17
- ➂ 严重错误(fatal error, involuntary)
- ➃ 被其他进程杀死(killed by other process, involuntary)
    - ➀ 收到一个信号 - unix chapter 10.2 p250 产生信号的方式

##### 1.4 进程的层次结构(process hierarchies)
当进程创建了另一个进程后，parent process和child process就以某种形式继续关联，child process可以创建更多进程，从而形成一个进程的层次结构(hierarchy)。但是，注意每个进程只有一个parent process，但可以有0、1个乃至多个child process。

▶︎ unix中的进程组(process group)概念
unix中，进程及其直接、简介创建的所有子进程构成一个进程组。当用户通过键盘发出一个信号时，该信号被传递给当前时刻与键盘输入相关的进程组(通常是当前窗口创建的所有活动进程)中的所有成员。收到该信号的所有进程可以分别捕获信号、忽略该信号、或者采取默认动作(即被该信号杀死)。

▶︎ 以unix系统初始化自身为例，理解进程层次
- ➀ 启动镜像(boot image)中有一个特殊进程，称为init(mac os中为launchd)。
- ➁ 当init开始运行后，读入一个终端(terminal)数量说明文件。
- ➂ 然后为每个terminal执行fork off创建一个新的进程(login process)。
- ➃ 这些被创建的进程等待someone执行登陆(login)操作，如果某个login操作成功，则login进程启动一个shell来接收someone的输入指令。
- ➄ someone在shell中键入的指令可能会创建出更多的进程，以此类推(so forth)。
- ➅ 最终，所unix系统中的进程都属于一颗进程层次树，init进程是该进程树的root节点。

▶︎ windows系统中没有进程层次的概念，所有进程地位相同 <br>
windows在创建进程时，parent process将得到一个speicial token(handle)，借助该token(handle)可以间接控制子进程。但是，parent process有权将token(handle)传递给其他进程，因而进程之间的层次概念被模糊了。

##### 1.5 进程的状态(process states)
每个进程都是一个独立的实体，有各自的程序计数器、内部状态；但进程之间通常需要相互作用：e.g. 一个进程的输入结果可能作为另一个进程的输入(在shell中键入指令)：
```txt
# first-process: cat指令将chapter1 chapter2 chapter3三个文件连接起来 & 输出 
# second-process: grep指令从输入中选择所有包含"tree"单词的那些行 & 输出
cat chapter1 chapter2 chapter3 | grep tree
# 上面指令运行结果 将chapter1~3三个文件合并的内容中所有包含"tree"单词的行打印出来
```
但是，根据这两个进程的运行相对速度不同(两个程序复杂度不同、分配到的CPU time不同)，上述程序运行结果是不稳定的。<br>
Consider：当grep准备就绪可以运行、但cat输入尚未得到结果时，就需要暂时"阻塞"grep直到得到输入。

▶︎ 进程发生阻塞的2种情况 <br>
➀ 当一个进程在逻辑上不能运行时，它就会被阻塞。这种情况下，进程被挂起是程序自身固有原因。<br>
➁ 由于操作系统调度另一个进程占用了CPU，一个逻辑上可以运行的进程被迫终止，它就会被阻塞。这种情况下，进程被挂起是系统资源技术上的原因。

▶︎ 进程的三种状态 <br>
运行态(running)：处于该状态的进程时刻占用CPU。<br>
就绪态(ready)：该进程可以运行，但因为其他进程正在运行而暂时停止。<br>
阻塞态(blocked)：除非外部事件发生使其运行，否则该进程不能运行，即使CPU处于空闲状态。

▶︎ 进程的三种状态之间的转换关系 <br>
<center><img src="/img/in-post/operating_system_img/os_2_4.pdf" width="90%"></center>
- ➀ 关于转换1：某些系统中进程可以执行a system call(e.g. pause)使进程进入阻塞状态。在其他系统(including unix)，当进程从pipe、special file(e.g. terminal)中读取数据，但读入数据不能作为可用输入时，该进程会自动进入阻塞状态。<br>
- ➁ 关于转换2/3：进程调度程序是操作系统的一部分，其主要作用是决定：❮运行哪个进程❯、❮何时运行❯、❮运行多长时间❯。关于调度程序，目前已经提出许多算法，主要解决：在❮efficiency for the system as a whole❯以及❮fairness to individual processes❯这对拮抗的要素之间权衡。
- ➂ 关于转换4：当进程等待的外部事件发生(e.g. 一些输入到达)，则该进程从阻塞态=>就绪态。如果它被调度程序选择，则立即发生转换3，否则它将在就绪态等待。

▶︎ 从"基于进程"的操作模型角度理解操作系统内部运行状态 <br>
下面展示了一个"基于进程"的操作系统概念图：操作系统最底层是调度程序，在调度程序上层有很多进程。
所有关于进程的中断处理、启动进程、进程停止的具体实现细节都隐藏在调度程序(scheduling)中；系统的进程的调度程序是一个非常简洁的程序。
操作系统中纷繁复杂的其他功能部分被组织成顺序进程(sequential processes)的形式。
<center><img src="/img/in-post/operating_system_img/os_2_5.pdf" width="70%"></center>
上面给出的概念图是一种❮理想中的模型❯，实际上很少有操作系统使用这种架构方式。

##### 1.6 进程机制的实现(process implementation) & 在中断过程中的切换&恢复
操作系统中维护着一个进程表(process table)，本质为一个结构数组，从而实现进程模型。<br>
每个进程占用一个进程表项，表项中包含了关于进程的重要信息：程序计数器(program counter)、堆栈指针(stack pointer)、内存分配状况(memory allocation)、打开文件状态(status of open file)、账号和调度信息(accounting & scheduling info)、以及所有在进程由运行态转换到就绪态或阻塞态时必须保存的信息，从而保证该进程再次启动时如同❮从未中断过一样❯。

▶︎ 进程表项中的关键字段 <br>
<center><img src="/img/in-post/operating_system_img/os_2_6.pdf" width="100%"></center>

▶︎ 单个CPU如何维护多个进程以实现"多进程同时运行"的错觉 <br>
CPU同一时刻只能运行一个进程，因此多进程不能真正在一个CPU中同时运行。CPU通过调度进程控制当前执行哪个进程、何时运行、以及该进程运行多长时间。CPU在不同的进程间快速切换，使得操作系统的用户认为CPU同时有多个程序处于运行态。

▶︎ 中断向量(interrupt vector) <br>
每个I/O类都和一个内存中的位置相关联(typically at a fixed location near the bottom of memory)，该存储空间被称为中断向量(interrupt vector)。中断向量包含了中断服务程序(interrupt service procedure)的地址。

▶︎ 程序计数器(program counter, PC) <br>
BIU => 译码器 => EU
PC和总线接口部件BIU相关，负责取指令：PC总是指向下一条待取指令，取出的指令经过译码后送至EU单元称为待执行命令。
CS:IP和执行单元EU相关，负责指向待执行指令：CS:IP总是被用于定位下一条待执行指令地址；顺序执行程序时，控制器首先根据CS:IP定位相应指令，然后分析&执行该指令，更新CS:IP内容以指向下一条将要执行的指令。<br>

▶︎ 系统进程中断发生后，操作系统底层工作流程(details may vary among systems) <br>
➀ 硬件(hardware)将当前进程寄存器状态(e.g. program counter...)压入堆栈(process table entry) <br>
➁ 硬件(hardware)从中断向量(interrupt vector)加载(load)新的程序寄存器状态(e.g. progrom counter) <br>
➂ 汇编语言程序(assembly-language procedure)保存所有寄存器状态，通常将寄存器状态保存在进程表项(process table entry)中 <br>
➃ 汇编语言程序(assembly-language procedure)设置新的堆栈(new stack used by the process handler)，即reset stack pointer。无论什么中断类型，saving register & set stack pointer的工作流程都是一致的，因此总是调用同一个assembly-language routine。<br>
➄ c语言中断服务程序(c interrupt service)在汇编语言例程完成后被调用，根据具体中断类型来完成the rest work(typically reads and buffers input)，c语言中断服务程序完成后，一般会将related process设置为就绪态(ready)，并调用调度程序(scheduler)。<br>
➅ 调度程序(scheduler)决定下一个将运行的进程。<br>
➆ c语言程序(c procedure)返回至汇编代码(assembly-code)，系统控制权交还给assembly-language code。<br>
➇ 汇编语言程序(assembly-language procedure)为新的current-process加载registers & memory map，并启动该进程进入运行态。<br>

系统进程在其执行期间可能会被中断上千次，在这种场景下保证进程正常运行的关键是：被中断的进程能够完全恢复到中断发生前的状态。

##### 1.7 多道程序设计模型(modeling multiprogramming)：一种unrealistically optimistic模型
使用多个进程可以提高CPU的利用率(utilization)，但是随着进程数量增加这种收益逐步减少。
<center><img src="/img/in-post/operating_system_img/os_2_7.pdf" width="100%"></center>
从完全精确描述进程和CPU利用率的模型构建，应该使用"排队法"完成模型的构建；但是概率模型在描述它们之间的关系时仍然是有效的(可以作为指定进程数、I/O等待时间比的条件下的CPU时间利用率的很好估计)。


## Reference
> \<modern operating system 4th\> chapter2 <br>
> \<unix环境高级编程 3ed\> chapter7.3 chapter8 chapter10.17 chapter 11.5<br>

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
