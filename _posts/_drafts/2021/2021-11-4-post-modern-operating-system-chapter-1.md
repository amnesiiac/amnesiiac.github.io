---
layout: post
title: "modern operating system: introduction"
subtitle: '[modern operating system] - chapter1' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-11-04 21:47
lang: ch 
catalog: true 
categories: operating-system 
tags:
  - Time 2021
  - modern operating system 
---
### 一：引言
##### 1 为什么需要操作系统
现代计算机是一个复杂的系统，如果每个程序员不得不掌握整个系统构造细节才能编写代码，则为计算机编写代码将是不可能完成的任务。另外，对于计算机系统组件进行管理&调用&优化同样是一项挑战性极强的工作。因此，计算机安装了一层软件，称为操作系统(operating system, os)。os的作用是为用户程序提供一个更好、更简单、更清晰的计算机模型，并借此管理计算机中的基础设施。

##### 2 计算机中软、硬件的基本组成部分结构简图
<center><img src="/img/in-post/operating_system_img/os_1_1.pdf" width="80%"></center>

##### 3 内核态和用户态
- 操作系统运行在**内核态(kernel mode)**。运行在内核态的os具有对所有硬件的的完全访问权，能够执行机器本身能够执行的任何指令(instruction)。
- 软件除了操作系统的部分运行在**用户态下(user mode)**。运行在用户态的软件只使用了机器指令的一个子集：那些会影响到机器控制、或者进行输入输出操作(I/O)的指令在用户态是禁止使用的。
- 嵌入式系统(embedded systems)可能不具有内核态；解释系统(interpreted systems)如基于java的系统通过解释的方式而不是硬件方式对计算机组件进行区分。
- 许多系统中，一些在用户态运行的程序协助操作系统完成一些需要特权的功能。这类系统中无法划分明显的界限。在内核态中运行的属于操作系统中的部分，但是一些在内核外运行的程序由于和操作系统密切相关，也可以被认为是操作系统的一部分(arguably part of os)。

##### 4 本文内容概要
本文主要介绍操作系统的若干重要组成部分，including它们的基本情况，它们的历史沿革，分类，以及操作系统中的一些基本概念及其结构。


### 二：什么是操作系统
##### 1 操作系统作为计算机体系结构的一种拓展(os as an extended machine) - top-down view
> **Why need os?** The architecture (instruction set, memory organization, I/O, and bus structure) of most computers at the machine-language level is primitive and awkward to program, especially for input/output.

> **(i) Operating systems contain many drivers for controlling I/O devices.** Clearly, no sane programmer would want to deal with this disk at the hardware level. Instead, a piece of software, called a **disk driver**, deals with the hardware and provides an interface to read and write disk blocks, without getting into the details. 

> **(ii) All operating systems provide yet another layer of abstraction for using disks: files.** Using this abstraction, programs can create, write, and read files, without having to deal with the messy details of how the hardware actually works.

> **This abstraction is the key to managing all this complexity.** Good abstractions turn a nearly impossible task into two manageable ones. The first is defining and implementing the abstractions. The second is using these abstractions to solve the problem at hand.

##### 2 操作系统作为资源管理器(os as a resource manager) - bottom-up view
> **从计算机资源管理角度来看，为什么需要操作系统？** When a computer (or network) has more than one user, the need for managing and protecting the memory, I/O devices, and other resources is even more since the users might otherwise interfere with one another. In addition, users often need to share not only hardware, but information (files, databases, etc.) as well. <br>
> **简而言之** In short, this view of the operating system holds that its primary task is to keep track of which programs are using which resource, to grant resource requests, to account for usage, and to mediate conflicting requests from different programs and users.

> **(i) 以自底而上的方式来解释操作系统对于资源费管理的作用** Modern computers consist of processors, memories, timers, disks, mice, network interfaces, printers, and a wide variety of other devices. In the bottom-up view, the job of the operating system is to provide for an orderly and controlled allocation of the processors, memories, and I/O devices among the various programs wanting them.

> **(ii) 操作系统进行资源管理时使用多路复用技术，包括时间上的复用&空间上的复用** Resource management includes multiplexing (多路复用, sharing) resources in two different ways: in time and in space. When a resource is time multiplexed, different programs or users take turns using it. First one of them gets to use the resource, then another, and so on.

##### 3 操作系统的历史(omitted)

##### 4 计算机硬件简介
<center><img src="/img/in-post/operating_system_img/os_1_2.pdf" width="100%"></center>

###### 4.1 处理器(processors)
> **CPU is the "brain" of the computer**. It fetches instructions from memory and executes them. <br>

**[1] CPU的基本流程循环** 
> The basic cycle of every CPU is to fetch the first instruction from memory, decode it to determine its type(类型) and operands(操作数), execute it, and then fetch, decode, and execute subsequent instructions. The cycle is repeated until the program finishes.

**[2] 关于CPU的指令集** 
> Each CPU has a specific set of instructions that it can execute. Thus an x86 processor cannot execute ARM programs and an ARM processor cannot execute x86 programs. 

**[3] 关于CPU中的通用寄存器**
> **为什么需要在CPU中使用寄存器** Because accessing memory to get an instruction or data word takes much longer than executing an instruction, all CPUs contain some registers inside to hold key variables and temporary results.
> **指令集的基本功能** The instruction set generally contains instructions to load a word from memory into a register, and store a word from a register into memory. 

**[4] 关于CPU中的几个对程序员可见的专用寄存器** 
> In addition to the general registers used to hold variables and temporary results, most computers have several special registers that are visible to the programmer.  <br>
> **(i) 程序计数器** One of these is the **program counter**, which contains the memory address of the next instruction to be fetched. After that instruction has been fetched, the program counter is updated to point to its successor. <br>
> **(ii) 堆栈指针** Another register is the **stack pointer**, which points to the top of the current stack in memory. The stack contains one frame for each procedure that has been entered but not yet exited(该堆栈包含了程序每个执行步骤的"栈帧"). A procedure’s stack frame holds those input parameters, local variables, and temporary variables that are not kept in registers("栈帧"保存了和程序执行相关的参数、本地变量、临时变量信息). <br>
> **(iii) 程序状态字寄存器** Yet another register is the **PSW (Program Status Word)**. This register contains the condition code bits(条件判断bit位), which are set by comparison instructions(被比较操作指令设置), the CPU priority(CPU优先级), the mode (user or kernel)(软件运行状态：用户态or内核态), and various other control bits(以及其他控制bit位). User programs may normally read the entire PSW but typically may write only some of its fields. The PSW plays an important role in system calls and I/O.

**[5] 关于CPU的3种架构的讨论** <br> 如下图所示，展示了3种CPU的架构组织方式，(a)表示一个将所有功能集成在一起的CPU；(b)表示CPU的流水线架构；(c)表示superscalar CPU架构。 

<center><img src="/img/in-post/operating_system_img/os_1_3.pdf" width="100%"></center>

> **(1) 由于性能上的原因，简单集成架构(a)很早就被放弃使用** To improve performance, CPU designers have long abandoned the simple model of fetching, decoding, and executing one instruction at a time(简单集成架构一次只能处理一条机器指令). <br>
> **(2) CPU的流水线架构(b)** Many modern CPUs have facilities for executing more than one instruction at the same time. For example, a CPU might have separate fetch, decode, and execute units, so that while it is executing instruction n, it could also be decoding instruction n + 1 and fetching instruction n + 2. Such an organization is called a pipeline. <br>
> **(3) CPU流水线架构带来的问题** **(i)** In most pipeline designs, once an instruction has been fetched into the pipeline, it must be executed, even if the preceding instruction was a conditional branch that was taken(一旦一个指令被读取到流水线中，它必须被执行，即使该指令前面是条件判断执行语句). **(ii)** Pipelines cause compiler writers and operating system writers great headaches because they expose the complexities of the underlying machine to them and they have to deal with them(流水线CPU架构将机器本身硬件底层的复杂性暴露给编译器作者、操作系统作者). <br>
> **(4) 超标量CPU架构的特性** In this design, multiple execution units are present(同时执行多个机器指令), e.g., one for integer arithmetic, one for floating-point arithmetic, and one for Boolean operations. Two or more instructions are fetched at once, decoded, and dumped into a holding buffer until they can be executed(≥2条机器指令被同时读取、解码并送到保持缓存区等待执行). <br>
> **(5) superscalar CPU架构带来的问题** An implication of this design is that program instructions are often executed out of order(该架构执行机器指令经常失序). For the most part, it is up to the hardware to make sure the result produced is the same one a sequential implementation would have produced(大多数情况，依赖硬件来保证执行次序不失序), but an annoying amount of the complexity is foisted onto the operating system, as we shall see(但也由于这种CPU架构而增加了os实现中的复杂度).

###### 4.2 多线程(multithreading) & 多核芯片(multicore chips)
**[1] 关于Moore定律**
> **(i) 定义** Moore’s law states that the number of transistors on a chip doubles every 18 months. <br>
> **(ii) Moore定律的解释** This "law" is not some kind of law of physics, like conservation of momentum, but is an observation by Intel cofounder Gordon Moore of how fast process engineers at the semiconductor companies are able to shrink their transistors. <br>
> **(iii) Moore定律的极限** Moore’s law has held for over three decades now and is expected to hold for at least one more. After that, the number of atoms per transistor will become too small and quantum mechanics will start to play a big role, preventing further shrinkage of transistor sizes.

**[2] 如何利用逐渐增加的晶体管资源？**
> **(i) Bigger caches** One obvious thing to do is put bigger caches on the CPU chip. That is definitely happening, but eventually the point of diminishing returns will be reached. <br>
> **(ii) More complicated control logics** The obvious next step is to replicate not only the functional units, but also some of the control logic. The Intel Pentium 4 introduced this property, called ❮multithreading❯ or ❮hyperthreading❯. <br>

**[3] 关于多线程**
> **(i) 什么是多线程？** Multithreading allows the CPU to hold the state of two different threads and then switch back and forth on a nanosecond time scale(CPU保存2个不同的线程状态，并在纳秒级时间内实现切换). A thread is a kind of lightweight process, which, in turn, is a running program(线程是一种轻量级进程，进程表示一个正在运行的程序，在chapter2中进行详细的介绍). <br>
> **(ii) 多线程之于操作系统** Multithreading has implications for the operating system because each thread appears to the operating system as a separate CPU. Consider a system with two actual CPUs, each with two threads. The operating system will see this as four CPUs.  

**[4] 关于多核芯片**
> Beyond multithreading, many CPU chips now have four, eight, or more complete processors or cores on them. The multicore chips as the following figure effectively carry four minichips on them, each with its own independent CPU. 

<center><img src="/img/in-post/operating_system_img/os_1_4.pdf" width="80%"></center>

> **多核芯片需要多处理器操作系统进行支持** Making use of such a multicore chip will definitely require a multiprocessor operating system.
> **上图中的intel架构和AMD架构各有千秋** Each strategy has its pros and cons. For example, the Intel shared L2 cache requires a more complicated cache controller but the AMD way makes keeping the L2 caches consistent more difficult.

**[5] 关于GPU**
> A GPU is a processor with, literally, thousands of tiny cores(微核心). They are very good for many small computations done in parallel(擅长大量简单并行计算), like rendering polygons in graphics applications. They are not so good at serial tasks(不擅长串行任务). They are also hard to program. <br> 
> While GPUs can be useful for operating systems (e.g., encryption(加密技术) or processing of network traffic(处理网络流量)), it is not likely that much of the operating system itself will run on the GPUs(操作系统本身不太可能运行在GPU上).


###### 4.3 存储器(memory)
<center><img src="/img/in-post/operating_system_img/os_1_5.pdf" width="80%"></center>

**[1] 关于CPU中的寄存器(register)**
> They are made of the same material as the CPU and are thus just as fast as the CPU. Consequently, there is no delay in accessing them.
> The storage capacity available in them is typically 32 × 32 bits on a 32-bit CPU and 64 × 64 bits on a 64-bit CPU. Programs must manage the registers (i.e., decide what to keep in them) themselves, in software.

**[2] 关于"memory word的定义"**
> A group of memory bits in a RAM or ROM block. Ref.[intel.com](https://www.intel.com/content/www/us/en/programmable/quartushelp/17.0/reference/glossary/def_memory_word.htm)。

**[3] 关于高速缓存(cache memory)**
> **高速缓存的基本结构** Main memory is divided up into cache lines, typically 64 bytes(64×8cache lines), with addresses 0 to 63 in cache line 0, 64 to 127 in cache line 1, and so on. 

> **高速缓存的作用** The most heavily used cache lines are kept in a high-speed cache located inside or very close to the CPU(架构设计上，将最常用的的cache集成在CPU的内部or放置在CPU临近处). When the program needs to read a memory word, the cache hardware checks to see if the line needed is in the cache(当程序需要读取内存字时，缓存相关硬件优先在cache中查找该字). If it is, called a cache hit, the request is satisfied from the cache and no memory request is sent over the bus to the main memory(如果在高速缓存中找到则称为缓存命中，因而无需通过总线发送memory request，从而提升CPU运行速率). 

> **限制高速缓存容量的因素** Cache memory is limited in size due to its high cost. 

> **高速缓存的解决的问题** **(i)** When to put a new item into the cache. **(ii)** Which cache line to put the new item in. **(iii)** Which item to remove from the cache when a slot is needed. **(iv)** Where to put a newly evicted item(新移除的项目) in the larger memory. 

> **L1高速缓存** The first level or L1 cache is always inside the CPU and usually feeds decoded instructions into the CPU’s execution engine(L1高速缓存通常在CPU内部，用于保存送往CPU执行引擎的机器指令). Most chips have a second L1 cache for very heavily used data words(部分芯片拥有第二块L1缓存). The L1 caches are typically 16 KB each. <br>
> **L2高速缓存** In addition, there is often a second cache, called the L2 cache, that holds several megabytes of recently used memory words(L2高速缓存用于保存近期使用的内存字). <br>
> **L1和L2高速缓存之间的区别** The difference between the L1 and L2 caches lies in the timing. Access to the L1 cache is done without any delay, whereas access to the L2 cache involves a delay of one or two clock cycles.

**[4] 关于主存(main memory)** 
> **随机访问存储RAM(volatile)** This is the workhorse(主力) of the memory system. Main memory is usually called RAM (Random Access Memory). All CPU requests that cannot be satisfied out of the cache go to main memory.

> **只读存储ROM(non-volatile)** Unlike RAM, nonvolatile memory does not lose its contents when the power is switched off(断电后，非易失性存储不会丢失所存储的内容). ROM (Read Only Memory) is programmed at the factory and cannot be changed afterward(只读存储在生产时烧入程序，后续无法修改). It is fast and inexpensive. On some computers, the bootstrap loader used to start the computer is contained in ROM(系统引导loader存储在ROM中). Also, some I/O cards come with ROM for handling low-level device control(IO接口卡中也有低层次的ROM用于设备控制).

> **EEPROM & 闪存(both non-volatile)和ROM的区别** EEPROM (Electrically Erasable Programmable ROM) and flash memory are also non-volatile, but in contrast to ROM can be erased and rewritten. However, writing them takes orders of magnitude more time than writing RAM, so they are used in the same way ROM is, only with the additional feature that it is now possible to correct bugs in programs they hold by rewriting them in the field. <br>
> **闪存** Flash memory is also commonly used as the storage medium in portable electronic devices(在便携式电子设备中广泛使用). It serves as film in digital cameras(电子相机胶卷) and as the disk in portable music players(随身听的磁盘), to name just two uses. <br>
> **闪存的传输速度 & 使用寿命** Flash memory is intermediate in speed between RAM and disk. Also, unlike disk memory, if it is erased too many times, it wears out.

> **CMOS存储(volatile)** Yet another kind of memory is CMOS, which is volatile. Many computers use CMOS memory to hold the current time and date. The CMOS memory and the clock circuit that increments the time in it are powered by a small battery(CMOS存储及其时钟电路由电池驱动), so the time is correctly updated, even when the computer is unplugged. The CMOS memory can also hold the configuration parameters, such as which disk to boot from(CMOS存储包括一些配置信息e.g.系统从哪个磁盘启动).

**[5] 关于磁盘(disk)** <br>
> **机械磁盘和RAM之间的访问速率 & 容量对比** Disk storage is two orders of magnitude cheaper than RAM per bit and often two orders of magnitude larger as well. The only problem is that the time to randomly access data on it is close to three orders of magnitude slower. The reason is that a disk is a mechanical device.

<center><img src="/img/in-post/operating_system_img/os_1_6.pdf" width="100%"></center>

> **机械磁盘读取访问占用时间分析** **(i)** Moving the arm from one cylinder to the next takes about 1 msec. **(ii)** Moving it to a random cylinder typically takes 5 to 10 msec, depending on the drive. **(iii)** Once the arm is on the correct track, the drive must wait for the needed sector to rotate under the head, an additional delay of 5 msec to 10 msec, depending on the drive’s RPM. **(iv)** Once the sector is under the head, reading or writing occurs at a rate of 50 MB/sec on low-end disks to 160 MB/sec on faster ones.

> **固态硬盘** SSDs(Solid State Disks) are really not disks at all. SSDs do not have moving parts, do not contain platters in the shape of disks, and store data in (Flash) memory. The only ways in which they resemble disks is that they also store a lot of data which is not lost when the power is off.

> **简介虚拟内存技术** Many computers support a scheme known as virtual memory. This scheme makes it possible to run programs larger than physical memory by placing them on the disk and using main memory as a kind of cache for the most heavily executed parts(将物理内存映射到磁盘上，并将主存作为程序频繁执行部分所需的"高速缓存"). <br>
> **内存处理单元负责执行这种"飞速"地址映射** This scheme requires remapping memory addresses on the fly to convert the address the program generated to the physical address in RAM where the word is located. This mapping is done by a part of the CPU called the MMU (Memory Management Unit).

> **浅谈应用程序上下文切换** The presence of caching and the MMU can have a major impact on performance(内存处理单元和缓存技术对于程序性能影响深远). In a multiprogramming system, when switching from one program to another, sometimes called a **context switch**, it may be necessary to flush all modified blocks from the cache and change the mapping registers in the MMU(为了实现这种应用切换，可能需要将高速缓存中所有被应用修改的区块执行刷新操作，并修改MMU中应用的映射寄存器数值). Both of these are expensive operations, and programmers try hard to avoid them(上述操作代价很高). 

###### 4.4 I/O设备  


## Reference
> \<modern operating system 4th\> chapter1 <br>

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
