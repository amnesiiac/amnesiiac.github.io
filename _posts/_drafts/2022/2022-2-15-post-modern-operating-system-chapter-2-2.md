---
layout: post
title: "modern operating system: 进程&线程(2)"
subtitle: '[modern operating system] - chapter2 - thread' 
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
## 本章概述(2) - 线程
操作系统中最核心的概念就是进程(process)，它是操作系统对于正在运行的程序的一个抽象；操作系统的其他所有内容都是围绕进程概念展开的。

进程是操作系统提供最早的、最重要的抽象概念。即使物理上可用的CPU只有一个，借助这种抽象，也可以虚拟出多个virtual CPU，从而使计算机支持(伪)并发操作的能力。

本文将介绍进程、线程的概念。

### 二：线程(threads)：mini processes
传统操作系统中，每个进程都包含一个地址空间(address space)、一个控制线程(thread of control)。在允许的情况下，在同一个地址空间中以"准并行(quasi-parallel)方式"运行多个控制线程是"可取的(desirable)"；多个线程运行机制和相互独立的进程类似，不同之处在于threads共享同一块地址空间。

在本文后续部分，将重点讨论线程的各种situations & implications。

##### 2.1 线程的使用(thread usage)
▶︎ **已经有了进程模型为什么还需要线程机制(mini-processes)？**<br>
在许多应用程序中需要线程机制的主要原因是：单个进程内可能同时运行多个活动(activities)；部分活动进程随着时间推移可能进入阻塞态(blocked)。通过将应用程序解构(decomposing)成多个"准并行(quasi-parallel)"顺序线程(sequential thread)，整个应用程序模型架构可以设计的更加简洁。

➀ 正因为有了进程模型的抽象，才不必过多思虑关于中断(interrupts)、定时器(timer)、以及程序上下文切换(context switches)，只需关注并行进程内部实现即可。同理地，正因为有了线程模型的抽象，才能让应用程序中的并行实体拥有共享同一个地址空间、以及所有可用数据的能力；而这种共享能力正是多进程模型所无法表达的(对于应用程序可能是必须的)。<br>
➁ 线程比进程更加轻量级，因此线程的create & destroy比进程更加容易(i.e faster)。在许多操作系统下，创建线程比创建进程快10～100倍；当存在大量线程需要被快速修改时，"fast-create"特性非常重要。<br>
➂ 如果创建的thread都是CPU bound类型的(CPU时间占用率很高)，则多线程模型并不能获得整体性能上的增强；然而，当整体任务需要大量的computing & I/O操作时，使用线程模型可以允许这些操作(computing & I/O)重叠执行，因此可以提升程序性能。<br>
➃ 在多CPU操作系统中，多线程是有益的，在这样的系统中可以实现真正的并行(见chapter8)。

▶︎ **文字处理软件案例分析：多线程程序优势** <br>
考虑设计一个文字处理软件(word processor)，根据用户指令完成相应文字排版任务的场景：用户在一本800页的书中删除了第一页中的一个句子，经过检查确认该删除操作是正确的，他现在想跳转到该书的第600页上(searching for some phrase occuring only there)。

使用单线程模型设计该软件：执行第一个删除操作后，文字处理软件将执行对书中0-600页的内容进行重新排版(time-consuming)，之后才能显示第600页上的的内容。这样，用户键入跳转指令后，需要等待相当长一段时间才能看到第600页的内容。

使用多线程模型可以提升软件用户体验，考虑一个拥有三个线程的文字处理软件：用户交互线程(interface)、后台文字排版线程(computing)、后台磁盘备份线程(I/O)：<br>
当收到用户删除指令后，交互线程立即通知后台文字排版线程执行相应操作，并执行全书排版工作；此时，交互线程可以继续为用户完成简单交互操作(e.g. 滚动页面...)；收到用户跳转指令后，交互线程通知后台排版线程"异步"执行跳转操作；如果排版操作完成的够快的话，用户在键入跳转指令后，可以很快看到600页上的内容。另外，为了避免由于程序崩溃、系统崩溃、电力故障等导致用户文档编辑丢失，后台磁盘备份线程可以定时完成备份操作。

<center><img src="/img/in-post/operating_system_img/os_2_8.pdf" width="100%"></center>

单线程软件架构中，后面的任务需要等待前一个任务完成后才能进行，这种设计方式严重影响用户交互体验，例如在执行文档排版时，用户其他交互操作将被忽略。一种解决方案是，在用户执行交互命令时，对文档排版任务执行系统中断，以及时响应用户交互，完成交互后，再恢复排版任务；这种方案引入了较为复杂的中断驱动程序设计模型(interrupt-driven programming model)。另外一种方案是使用较为简单的3线程模型。

▶︎ **web服务器案例分析：多线程程序优势** <br>
➤ 1 多线程web服务器框架
<center><img src="/img/in-post/operating_system_img/os_2_9.pdf" width="85%"></center>
- web page cache和disk(database)存储：在某些站点上，其主页的访问次数远远多于页面树靠近叶子节点的网页，因此在设计web服务器时可以将大量访问的页面保存在内存中，便于快速调用，称为页面高速缓存(page cache)；其他较冷门页面在磁盘中构建的数据库中存储。
- dispatcher thread：从网络中读取请求，检查请求后，dispatcher挑选一个idle worker thread并向其传递该请求(possibly by writing a pointer to the message into a special word associated with each thread)。然后，dispatcher thread唤醒该sleeping worker thread，将该线程从阻塞态变成就绪态。
- worker thread被唤醒后，检查收到的请求是否在Web高速缓存中(该缓存被所有worker thread共享)；如果在，则直接从web高速缓存中调用结果进行应答；否则，worker thread从磁盘执行read操作读取请求的页面，并设置work thread为阻塞态直到disk I/O操作完成(系统调用将阻塞该calling thread，即这是一个阻塞的系统调用)。
<center><img src="/img/in-post/operating_system_img/os_2_10.pdf" width="80%"></center>

➤ 2 单线程web服务器框架
<center><img src="/img/in-post/operating_system_img/os_2_11.pdf" width="70%"></center>

➤ 3 非阻塞系统调用 & 有限状态机模型构建web服务器框架
<center><img src="/img/in-post/operating_system_img/os_2_12.pdf" width="95%"></center>
- 在如上图中展示web服务器程序框架中，没有使用如2/3中的"顺序进程模型(sequential processes)"。每次web-server从为某个请求工作的状态切换到另一个状态时(为新任务请求服务/返回旧任务请求的应答)，都需要保存/重新加载计算状态；详见"\*"标识的内容。
- 实际上，上述程序框架相当于以一种困难的方式模拟了线程、及其堆栈的底层操作。
- 另外，上述程序框架中CPU的每次计算的结果、环境都被抽象成一个状态；进而，处理web页面请求、cache/disk执行I/O中的操作流程可以抽象成一系列允许发生、并引起相关状态改变的状态集合；这种抽象方式称为有限状态机模型(finite-state machine)。有限状态机的模型的抽象方式在计算机科学中被广泛使用。

➤ 4 结合1~3的案例，理解"顺序进程"思想 <br>
➀ 多线程模型利用了顺序进程的思想(sequential processes)：即阻塞了系统调用，但是实现了请求处理时的并行性。简言之，对系统调用(上例中system call:read)的阻塞，使得程序设计得以简化，并提升了程序的整体性能。多线程模型以编程的复杂性代价换来了相对较大的灵活度。<br>
➁ 单线程模型无法实现请求处理时的并行性，当执行系统调用处理当前请求时，后续到来的请求被阻塞，直到系统调用(e.g. system call:read)完成对当前请求的处理(系统调用此时也以阻塞方式运行)。<br>
➂ 有限状态机模型可以实现请求处理的并行性，并且每个请求处理引起的系统调用(e.g. system call:read)、系统中断之间以非阻塞方式运行。有限状态机模型以编程的复杂性为代价换来了最大的灵活度。<br>
<center><img src="/img/in-post/operating_system_img/os_2_13.pdf" width="90%"></center>

➤ 5 关于多线程程序应用场景的思考 <br>
多线程程序非常适合那些必须处理大量数据的应用。<br>
首先，考虑"single thread model"：读取一块数据 -> 对数据进行处理 -> 执行处理结果写出。显然，该模型在执行数据读取、写出时，会阻塞该线程(无法处理数据)，这样CPU将空转，当有大量计算需要处理时，是计算资源的极大浪费。然后，考虑"multi thread model"使用3个独立的线程来完成：读取数据线程、数据处理线程、结果数据写出线程。
<center><img src="/img/in-post/operating_system_img/os_2_14.pdf" width="90%"></center>
按照上图模型工作方式，数据读取、数据处理、结果写出可以同时进行。<br>
**当然，需要注意多线程模型正常工作的前提**：系统调用只是阻塞调用它的线程(calling thread)，而不是阻塞调用它的整个进程时(该进程包含了多个calling thread)，才能正常工作。

##### 2.2 经典的线程模型(the classical thread model)
在chapter2中第一篇文章中介绍的进程模型是基于2个独立的概念：资源分组(resourse grouping)、以及相应程序独立执行。在本文中介绍的线程概念则处理共享资源下的程序独立执行问题。

本小节将先介绍经典进程模型，然后再介绍"模糊了进程/线程分界线"的linux线程模型。

▶︎ **从资源(resources)角度理解线程、进程** <br>
进程是一种抽象概念，它将运行所需的相关资源集中在一起，包括：程序正文、相关数据、以及其他资源所构成的地址空间(详见下表)。这些资源通过进程的抽象方式予以管理。<br>
另外，每个进程还包含一个执行的线程。线程包含：程序计数器(PC)用于记录当前待取指令地址；寄存器用于记录当前工作变量(working var)；堆栈用来记录执行历史记录(execution history)：每个栈帧保存了一个已调用但尚未得到返回的过程。
**进程是把资源集中在一起并进行管理的抽象，线程是在CPU上被调度执行的实体**。

➤ 进程、线程模型及其地址空间在用户、内核空间中的模式图 <br>
<center><img src="/img/in-post/operating_system_img/os_2_15.pdf" width="90%"></center>
- 左图中3个线程分别属于不同的进程，因而拥有不同的地址空间；右图中3个线程属于同一个进程，它们共享同一个地址空间。
- 同一个进程中的不同线程共享地址空间，因而共享同一个全局变量，并且每个线程都可以访问地址空间中的每个内存地址，所以一个线程甚至可以读、写、甚至擦除另一个线程的堆栈。
- 同一进程内的线程之间是没有保护的，原因是：1) 不可能 2) 没必要。不同的进程来自不同的用户，因此它们之间存在资源竞争、用户优先级竞争关系；但同一进程下的不同线程属于同一用户，用户创建多线程的目的本身应该为了它们彼此合作而不是拮抗。<br>
因此，对于没有关系的线程，应该按照左图划分到不同进程中进行处理；对于彼此合作关系的线程，则应按照右图方式进行处理。

➤ 进程、线程管理的计算机资源 <br>
<center><img src="/img/in-post/operating_system_img/os_2_16.pdf" width="70%"></center>

▶︎ **线程的基本状态** <br>
同传统进程(i.e. 只含单一线程的进程)一样，线程可以处于如下几种状态任意一个：运行态(running)、阻塞态(blocked)、就绪态(ready)、终止态(terminated)。线程各种状态之间的转换和进程是一样的。
<center><img src="/img/in-post/operating_system_img/os_2_17.pdf" width="90%"></center>

▶︎ **与线程相关联的堆栈 & 一个特殊栈帧** <br>
<center><img src="/img/in-post/operating_system_img/os_2_18.pdf" width="80%"></center>
- 每个线程的堆栈中都有一个特殊栈帧(frame of stack)，供每个被调用但尚未得到返回的procedure所使用。该栈帧包含了该procedure的局部变量(local var)，以及该procedure调用完成后使用的返回地址。
- 例如，X过程调用Y过程，Y过程调用Z过程；那么当过程Z执行时，此时X、Y、Z过程都处于未返回状态，则对应线程的特殊栈帧中将包含X、Y、Z三个过程的局部变量、调用完成的返回地址。

▶︎ **线程的创建/等待/切换/销毁 & creating thread和created thread的关系** <br>
➀ **创建** 多线程并行的场景通常从一个初始线程开始，该线程通过调用库函数(e.g. thread_create)来完成线程的创建，并且通过库函数参数指定了被创建线程所调用的过程(procedure)。creating thread通常返回一个"线程标识符(identifier)"作为created thread的名称。<br>
需要注意，在创建线程时，关于被创建线程的地址空间无需(甚至不能)做任何假设，因为被创建的线程自动在creating thread所在的地址空间中运行。
另外，有时线程之间存在层次关系(parent/child)，但通常情况下不存在这种关系，所有的线程都是平等的。<br>
➁ **等待** 另外，通过调用库函数(thread_join)，线程可以等待一个指定线程退出；该库函数阻塞calling thread直到指定的线程退出；thread_join用于如下场景："线程A的后续执行需要等待线程B执行完成才能进行"。<br>
➂ **切换** 通过调用库函数(thread_yield)，可以让当前线程放弃占用CPU，从而让另一个线程开始执行。这个功能十分重要，进程可以使用时钟中断强制进程放弃CPU占用，线程库无法做到这一点，因此在必要时让线程主动交还CPU(voluntarily surrender)，允许其他线程执行是非常重要的。<br>
➃ **销毁** 当线程完成工作时，可以调用库过程(e.g. thread_exit)退出，于是该线程被销毁，不能被调度程序调用。

▶︎ **线程为程序的编写带来复杂性** <br>
通常情况下，使用线程是有益的，但是使用线程为程序实现带来了一定复杂性：

➤ 线程在parent/child process中的实现细节思考 - problems related to the fork call <br>
考虑unix系统中用于创建child process的fork系统调用，如果parent process拥有多个线程，那么child process是否也应拥有这些线程？<br>
如果child process不拥有这些线程，则child process可能不能正常运行：进程目标被解构成不同的线程处理，每个线程都是必要的。<br>
如果child process拥有和parent process一样的线程，则当parent process在执行中被系统调用(e.g. system call:read)所阻塞，则child process是否也同样被该系统调用(e.g. system call:read)阻塞？假设parent/child process调用[system call:read]读取键盘输入，则parent/child process是否同时阻塞在该键盘上以监听输入信息？在键盘上完成输入后，parent/child process都会得到该输入信息的副本么？还是只有parent/child其中之一得到该信息副本？

➤ 线程共享地址空间中的数据结构所带来实现细节思考 - problems related to the share on data structures <br>
如果一个线程关闭了另一个线程正在读取的文件会发生什么？又或者如果一个线程发现申请的内存空间太少，并开始分配更多内存时，此间(partway through)，执行了一次线程切换，新切换到的线程也发现内存空间过少，并开始分配更多内存；在2个进程切换过程中，可能发生了2次内存分配。

[总结] 正确的处理线程上述问题，需要careful thought and design才能使多进程程序正确地工作。

##### 2.3 POSIX线程
为了便于线程程序的移植性，IEEE-1003.1c中定义了线程标准，被大部分unix系统所支持，它定义线程包称为pthread。所有pthread线程都具有的属性包括：每一个都有标识符、一组寄存器(PC...)、一系列属性(堆栈大小、调度参数、其他的运行线程所需items)，列举pthread主要函数如下表：
<center><img src="/img/in-post/operating_system_img/os_2_19.pdf" width="70%"></center>
pthread相关函数的用法和chapter-1中进程机制类似。

##### 2.4 在用户空间中实现线程
有2种主要方法实现线程机制：在用户空间中实现线程、在内核空间中实现线程；另外，同时借助用户、内核空间实现线程也是可行的(hybrid)。这2种方法各有利弊，下面分别介绍这两种方式，并分析它们的优缺点。

▶︎ **在用户空间中实现线程 - user-level thread package** <br>
这种方式将线程实现函数库放在用户空间中，内核对于线程的情况一无所知；从内核角度考虑这种实现，等价于内核将线程程序当作普通程序一样进行管理，即将所有程序都作为单线程进程对待。这种实现方式的明显的优势是：用户级线程包(user-level thread package)可以在不支持多线程的操作系统上实现。

下面给出一个使用用户级线程包的程式例子：
```cpp
#include <pthread.h>// include pthread package
#include <stdio.h>
#include <stdlib.h>

#define NUMBER_OF_THREAD 10
void* print_hello_world_and_exit(void *tid){// print & exit thread
    // output the thread identifier 
    printf("Hello World. Greetings from thread %d\n", tid);
    pthread_exit(nullptr);// c++(nullptr -> void*)
}
int main(){
    pthread_t threads[NUMBER_OF_THREAD];// thread arr 
    int status;
    for(int i=0; i<NUMBER_OF_THREAD; i++){
        printf("Main here. Creating thread%d\n", i);
        // create thread... 
        status=pthread_create(&threads[i], nullptr, print_hello_world_and_exit, (void*)i);
        if(status!=0){// check
            printf("Oops. pthread_create returned error code %d\n", status);
            exit(-1);// abnormal exit
        }
    }
    exit(NULL);// normal exit(NULL=0 in c++)
}
```

▶︎ **用户级、内核级线程实现中，线程表、进程表的配置方式** <br>
<center><img src="/img/in-post/operating_system_img/os_2_20.pdf" width="100%"></center>
在用户空间管理线程时，每个进程都需要维护一个线程表(thread table)，用于跟踪进程中每个线程的状态。线程表的实现同内核空间中的进程表(process table)类似，不过它只需记录每个线程属性即可，包括：线程的PC、堆栈指针、寄存器、线程状态等。<br>
线程表由运行时操作系统(run-time system)管理。当一个线程转换到就绪态or阻塞态时，重启该线程的信息被存入线程表对应位置；和内核在进程表中保存进程状态的机制如出一辙。

➤ 关于运行时系统(run-time system)的含义可以理解为：已编译好的程序在运行时所需的支持系统。

▶︎ **用户级线程间的切换 & 相比内核级线程的优势 & 用户级线程存在的问题** <br>
➤ 用户级线程实现的调度流程举例：<br>
当某线程执行了一些引起自身阻塞的任务后(e.g.调用thread_join等待其他进程退出)，它将调用一个运行时系统下程式(procedure)，以检查该线程是否必须进入阻塞态。<br>
如果必须进入阻塞态，该线程在线程表中保存自身寄存器(var)，查询线程表中处于就绪态的线程，井把新线程在线程表中保存的寄存器内容加载到(reload)计算机的寄存器中(machine registers)。一旦stack pointer和program counter完成切换，新进程自动恢复执行。

➤ 用户级线程实现相比内核级线程实现的优势：<br>
➀ 不同于进程(进程表项总是在内核操作系统中实现)，线程在完成运行&执行线程切换时(e.g. 调用thread_yield)，thread_yield实现代码负责将线程相关信息记录在线程表中(用户级线程包在run-time system中实现线程表)，因此旧线程的保存、新线程的切换都是"本地过程"，不需要深入内核中进行操作，不需要进行上下文切换，不需要刷新内存中高速缓存。因此，这就是为什么用户级线程的调度更加快捷的原因。<br>
➁ 用户级线程实现的另一个优势是：允许每个进程自定义内部线程的调度算法。在那些具有垃圾回收(garbage collection)线程的应用程序中无需担心线程在不合适的时候停止(自定义调度gc线程以实现资源回收释放)。<br>
➂ 用户级线程的实现的第3个优势是：具有较好的可扩展性。内核级线程实现中，线程需要占用固定的表格空间、堆栈空间，如果内核线程数量很多，可能会引发问题；而用户级线程占用的空间可以自定义，因此扩展性更好。

➤ 用户级线程实现存在的明显问题：<br>
➀ 用户级线程不太容易实现❮阻塞系统调用❯问题：在没有任何按键发生之前，使用一个线程监听键盘，在该线程内部执行系统I/O调用是不可接受的，因为❮阻塞系统调用❯会阻塞管理该线程的进程，从而导致该进程中所有线程停止运行。<br>
solution-1：将阻塞系统调用改成非阻塞系统调用，比如，当键盘缓存区没有按键字符信息时，线程读取键盘操作将返回0 bytes，于是此线程对应的进程不需要阻塞在键盘上监听下一个输入；但是需要改写操作系统底层实现，修改[system call:read]的语义将导致其他应用程序兼容性问题，而用户线程具有的操作系统适应性优势不复存在，因此不推荐此方案。 <br>
solution-2：在某些unix os下，使用[system call:select]可以允许通知预期[system call:read]是否会发生阻塞；于是用户级线程可以先借助select系统调用判断是否预期发生阻塞，如果不会阻塞才执行read系统调用，否则运行其他线程，下次运行时系统取得该线程控制权时，再次判断执行read系统调用是否"安全"。这种方案需要重写系统调用库，是一种inefficient & inelegent的方法，但是几乎没有其他备选方案了。借助系统调用代码并加以修饰以执行特定检查的代码称为"包装器(wrapper)"。

➁ 用户级线程的❮缺页故障引起阻塞系统调用❯问题：计算机被设定成以如下工作模式运转：同一时刻没有将所有程序放入内存中；如果程序调用(program call)、程序跳转(program jump)指向了另一个没有加载在内存中的程序时，操作系统将会从disk中获取内存中确实的指令部分；称之为缺页故障(page fault)。当系统I/O调用从disk中对缺失的指令进行定位、读取时，将阻塞相关进程执行：某个线程引起缺页故障，内核不知道线程存在，通常将整个进程阻塞直到系统I/O完成，尽管缺页问题不会影响其他线程运行。

➂ 用户级线程的❮一个线程开始运行后，其他线程将不能执行，除非当前处于运行态的线程主动放弃CPU占用❯问题：在同一个进程下，无法实现时钟中断(clock interrupt)，因此无法调度该进程下的线程实现轮转执行(round-robin fashion)，除非一个线程可以主动进入运行时系统并执行，进程下调度程序无法控制线程主动执行。<br>
solution-1：让运行时系统每秒请求1次时钟信号(中断)，赋予运行时系统每秒1次执行时钟中断的能力，但这种中断对于线程中正在执行的程序是生硬、无序的，这种高频率周期性系统时钟中断严重伤害程序的执行、且代价也很大。更进一步讲，运行时系统中被中断的线程可能也在运行一个时钟中断请求过程，运行时系统高频率周期性执行的时钟中断可能会和线程内部的时钟中断相互干扰。

➃ 即使用户级线程❮阻塞系统调用❯很难实现，并且可能由于❮缺页故障引起阻塞系统调用❯，具有❮同进程下的线程调度很棘手❯等问题，但是程序员几乎只在那些❮经常发生阻塞式系统调用的场景下❯才使用用户级线程(e.g. multi-threaded web-server)。<br>
但是，在经常发生阻塞式系统调用的场景下，实现线程切换也不是容易的事：一旦某用户级线程请求内核执行了一个"陷阱系统调用"(该系统调用需要阻塞调用线程很长时间才能获得输入)，则内核将几乎无法完成线程切换。另外，让内核负责此类线程的切换和"wrapper机制"功能相互重叠(redundant)：用户级线程定期借助[system call:select]预估[system call:read]是否发生阻塞。
对于那些本身CPU占用率很高、极少发生阻塞的应用程式而言，没有引入线程架构的必要：e.g. 没有人会提出使用多线程模型来完成下象棋一类的任务。

##### 2.5 在内核空间中实现线程    
考虑内核支持、管理线程的情形，此时不需要运行时系统了。另外，管理线程、进程的线程表、进程表都在内核中实现，因此当有线程需要创建新线程、撤销已有线程时，将执行一个系统调用，该调用通过更新线程表来完成线程的创建、撤销。现在由内核维护、跟踪每个线程、进程的状态信息。

➤ 内核级线程相比用户级线程在thread managment上差异分析：<br>
内核级线程的阻塞都将以系统调用的形式实现，因此相比借助运行时系统过程(procedure)来管理线程，内核级线程的阻塞将会产生更大的代价。这种代价换来了相对用户级线程的灵活性：<br>
当某个内核级线程被阻塞，内核可以选择执行同一进程下的另一线程、或者执行其他进程下的线程。<br>
然而，对于用户级线程，运行时系统只能维护当前进程下的线程运行，直到内核剥夺了它的CPU资源、或者该系统下所有线程执行完毕，才能切换到其他进程下执行。

➤ 内核级线程管理中的thread recycling机制：<br>
由于创建、销毁内核级线程的巨大开销，一些操作系统采取了相对"环保"的方式循环利用(recycle)它们管理的线程：当某线程被销毁，内核只是将其标记为"not runnable"，并不会销毁其占用的内核数据结构；后续如果新的线程需要被创建，则借助一些废弃线程在内核空间中的资源，可以节省一些overhead。<br>
thread recycling操作在用户级线程中也能实现，但由于用户级线程的管理overhead本身较小，通过不会实现该机制(less incentive)。

➤ 内核级线程相比用户级线程的优势 <br>
➀ 内核级线程不需要任何新的、非阻塞式系统调用。<br>
➁ 内核级线程可以很好地应对页面故障问题：由于内核级进程管理的灵活性，当程序调用、程序跳转指向了一个没有加载在内存中的程序时(page fault)，内核可以方便的检查该进程下其他可运行线程，并选择执行。内核管理线程的灵活性是以一定开销为代价换来的：如果内核级线程的创建、中止等操作较频繁发生，开销将会非常大。

➤ 内核级线程并不能cover掉所有问题 <br>
➀ 当多线程进程创建一个新进程时，新进程是否拥有原进程相同数量的线程，还是只有一个线程？这取决于该进程将要执行的动作：如果该进程要调用exec创建一个新的程序，则新进程拥有一个线程是最好的选择；如果该进程创建新进程后继续执行，则最好复制(reproducing)全部线程。<br>
➁ 信号是发送给进程而不是线程的，那么当多线程进程收到一个信号，应该由哪个线程负责处理它？线程可以向进程注册它"感兴趣"的信号，当信号到达进程后，将其交给需要它的线程。但是如果超过2个以上的线程都向进程注册同一个"感兴趣"的信号，那么问题将会变得复杂起来。

##### 2.6 混合用户空间、内核空间以实现线程
用户级线程、内核级线程各有千秋，因此本小节内容尝试将两者优点结合起来。<br>
使用内核级线程，并将用户级线程以❮多路复用multiplex❯的方式继承自some/all of这些内核进程，如下图所示：
<center><img src="/img/in-post/operating_system_img/os_2_21.pdf" width="75%"></center>
- 通过multiplex，程序员可以自定义需要的内核级线程数，以及每个内核级线程可以复用成多少个用户级线程；这个模型为此提供了最大的灵活度。
- 通过multiplex，内核将只负责内核级线程部分(管理、调度)，部分内核级线程上以多路复用的方式派生出多个用户级线程；这些用户级线程的创建、销毁、调度方式，同在不具有多线程支持的OS上进程中的用户级线程一样。
- 通过multiplex，每个内核级线程都被拥有一组用户级线程轮流使用。

##### 2.7 调度线程激活机制(scheduler activations) - 结合用户、内核线程优势
尽管内核线程在一些关键点上优于用户级线程，但是内核线程的管理调度较用户级线程而言慢得多。因此，希望找到保持内核线程优良特性的前提下，改善其管理、调度速度的方法：调度线程激活机制。

调度线程激活机制主要的思路是：模拟内核级线程的功能，但是为线程包提供了用户级线程具有的性能、灵活性。细节方面：<br>
➀ 如果用户线程在执行某些系统调用为"安全的(系统调用不阻塞调用线程)"，则不应该进行"专门设计的"非阻塞调用、以及进行提前"安全性检查"。<br>
➁ 如果线程阻塞在某个系统调用、页面故障上，只要同一进程中有其他就绪态进程，则这些线程可以得到调度执行(模拟内核级线程)。

关于调度线程激活机制实现的细节，待查阅相关资料进行整理补充。

##### 2.8 弹出式线程(pop-up threads) - 处理系统外到来的信息
在分布式系统(distributed systems)中经常使用线程模型，其中一个经典应用案例是：如何在分布式系统中利用线程处理到来的信息(incoming messages)？

传统的方式是将进程、线程阻塞在一个receive系统调用上，等待到来的信息；当信息到达后，该系统内接受调用信息，打开并检查信息内容，然后进行处理。

对于上述场景，还可以使用弹出式线程的方式完成：外部信息的到达，使得分布式系统创建一个处理该信息的线程(弹出式线程)。这种方式的优势在于：系统创建的弹出式线程非常新(brand new)，线程历史状态简单易维护：寄存器、堆栈等都处于fresh状态；因此，创建这种类型的线程所需时间很短，即外部信息从到达到处理的时间非常短。
<center><img src="/img/in-post/operating_system_img/os_2_22.pdf" width="90%"></center>
但是在使用弹出式线程接受系统外部信息之前，需要决定弹出式线程的级别(内核级or用户级)？<br>
如果系统内核上下文(kernel's context)支持运行弹出式线程，则弹出式线程可作为内核级线程实现。相比用户级线程，在内核空间实现的弹出式线程可以更加快速、便捷地访问相关表(kernel tables)以及I/O设备；这种特性在系统中断处理时是有用的。<br>
但是，当内核级弹出式线程出错时，相比用户级线程，代价更大。例如：当一个无效的内核级线程运行时间太长，又无法抢占(preempt)它以执行其他线程，就可能造成新到来的外部信息丢失。

##### 2.9 使单线程代码多线程化(making single-threaded code multithreaded)
许多程序是以单线程模式来编写的，将这些程序改写成多线程相比直接书写多线程模式程序更加棘手(trickier)。考虑下面的几种在代码改写过程中易犯的错误(pitfalls)：待整理。

线程代码内部同样包含多个过程，并且拥有全局变量、局部变量、以及过程参数。局部变量和过程参数在单线程代码改写成多线程代码时不会有什么问题，它们分别属于自己的线程部分；但是全局变量需要特别注意：许多全局变量之所以是全局的，是因为线程中的许多过程(e.g. functions)需要调用它们，它们本身和多线程程序的逻辑无关(每个线程应拥有它的一个副本，而不是共享同一全局变量)。

▶︎ **以unix系统error变量为例，考虑程序改写注意事项** <br>
待整理。



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
