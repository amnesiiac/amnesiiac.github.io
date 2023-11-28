---
layout: post
title: "modern operating system: 进程&线程(4)"
subtitle: '[modern operating system] - chapter2 - scheduling' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2022-03-01 20:57
lang: ch 
catalog: true 
categories: operating-system 
tags:
  - Time 2022
  - modern operating system 
---
## 本章概述(4) - 调度 
当计算机系统为支持多道程序的系统时，每当2个或以上的进程处于就绪态时，将出现多个进程/线程同时竞争CPU资源的情形。为了保证多个进程看起来好像在同时运行，CPU必须快速完成进程/线程的切换；操作系统中选择下一个将执行的进程的部分称为调度程序(scheduler)，相应的算法称为调度算法。

绝大多数情况下，适用于进程的调度处理方法也同样适用于线程。本文后续部分将首先介绍同时适用于进程/线程的调度方法，然后会单独介绍线程调度以及它所产生的独特问题。

### 四：调度(scheduling)
##### 4.1 调度简介(introduction)
调度算法程序的发展经历了几个典型阶段，每个阶段对于调度程序的需要不同；另外，在同一阶段的不同场景下，对调度程序也有不同的需求：

➀ __以磁带卡片作为输入的批处理系统__ 只需依次运行磁带上的每个作业，不需要调度算法。<br>
➁ __多道程序设计系统__ 由于经常有多个用户等待服务，对应的调度算法较为复杂。<br>
➂ __将批处理程序和分时服务程序结合使用的大型机系统__ 需要调度程序决定下一个运行的是批处理任务，还是终端用户互动任务。<br>
在➁、➂场景中，多个任务下的CPU属于"稀缺资源"，因此调度程序算法的好坏能够显著影响计算机执行任务的效率。

➃ __个人计算机系统__ 主要有2方面发展趋势：<br>
* ➃-➀ 多数时间中，个人计算机系统中只有一个活动进程(active process)；在这种较简单的进程调度场景下，调度程序的任务变得简单许多。<br>
* ➃-➁ 个人计算机系统硬件速度极快，相对处理任务数CPU资源已经不再稀缺；个人计算机的多数程序的运行主要受到用户键盘、鼠标输入速率的限制，而不是CPU处理速度的限制，因此调度程序的优化的意义不如前面的场景中有价值。

➄ __网络服务器系统__ 场景下调度程序很重要：<br>
在网络服务器系统下，经常发生多个进程竞争CPU资源的情况，CPU必须通过调度程序决定当前"最应该"处理的任务是什么，此时调度程序显得尤为重要。

➅ __移动设备系统__ 场景下，资源充足是伪命题：<br>
智能手机的计算资源总是多多益善，移动端CPU发展速度不能充分满足日新月异的软件需求，此时调度算法有较大的用处。另外，即便手机配备了先进的CPU，调度程序仍然有用武之地：用于优化移动设备的电力损耗。

▶︎ __进程行为(process behavior)__ <br>
几乎所有进程的 __I/O活动__ 和 __计算__ 都是交替发生的，例如：CPU执行运算持续一段时间，然后发出一个系统调用以便读写文件(该系统调用将阻塞调用进程)，在完成系统调用后，CPU又开始运算。<br>
一种用于划分 __I/O活动__ 和 __计算过程__ 的标准是：只要CPU参与的过程都属于计算，当进程等待外部调用而被阻塞时才属于I/O活动。这种阶段划分方式有助于分析CPU调用时序。

➤ CPU密集型进程、I/O密集型进程时序图
<center><img src="/img/in-post/operating_system_img/os_2_35.pdf" width="100%"></center>
随着计算机硬件逐步发展，CPU的计算速度越来越快，硬盘的I/O速度也在变快。但越来越多的进程趋向于I/O密集型，这是因为CPU的改进相比硬盘的改进要快得多。这种趋势将导致针对I/O密集型进程的调度处理显的尤为重要。对于I/O密集型进程而言，同时并行多个I/O密集型进程可以保持CPU资源的充分利用。

▶︎ __何时进行调度决策(when to schedule)__ <br>
➀ 在创建一个新进程后，需要决定继续运行父进程还是转而执行子进程(这两个进程此时都处于就绪态)。调度程序可以合法的选择任意一个进程执行。<br>
➁ 在一个进程退出时，需要进行调度决策。调度程序需要从就绪态进程集中选择一个执行，如果没有就绪进程则需要运行系统提供的空闲进程(idle process)。<br>
➂ 当一个进程阻塞在I/O上调用、或阻塞在信号量上(lock)、或由于其它原因阻塞时，必须选择其它进程执行。<br>
➃ 当I/O中断发生时，必须进行调度决策。如果I/O中断来自一个"空闲状态(job finished)"的I/O设备，则哪些阻塞以等待该设备I/O的进程将变为就绪态。<br>
➄ 当硬件时钟提供50hz/60hz以及其它频率的周期性中断(periodic interrupts)，则可以在每个时钟中断、或者每k个时钟中断做出调度决策。根据如何处理时钟中断，调度算法可以被分成2类：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➄-➀ 非抢占式算法(non-preemptive)：算法选择一个进程执行，该进程将保持运行，直到被阻塞or该进程主动释放CPU占用。因此，在时钟中断发生时，不会运行调度程序；在处理完时钟中断后，除非有更高优先级的进程处于等待超时状态，否则该进程将继续执行。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➄-➁ 抢占式算法(preemptive)：算法选择一个进程，设置最大运行时间(t_max)后执行该进程。如果超过最大运行时间后，进程仍在执行，则调度程序将其挂起，并挑选其它就绪态进程执行。这种抢占式调度处理需要在t_max后产生时钟中断，以将CPU的控制权交给调度程序。如果没有可用的时钟，则只能选择非抢占式调度流程。

▶︎ __调度算法分类(categoies of scheduling algorithms)__ <br>
不同的场景下需要不同的调度算法，这是因为：不同场景下进行CPU资源利用的优化时，限制因素不同。3种典型的场景如下：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➀ 批处理系统(batch) <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➁ 交互式用户环境(interative) <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➂ 有实时限制的系统(real-time) <br>
➀ 批处理系统在商业领域有广泛的应用，尤其是在一些处理周期性任务的场景中，在这类场景下，不会有用户在终端等待一个短请求的快速响应(quick response)。此类场景下，非抢占算法、每个进程都设置了长时间周期的抢占式算法，都有广泛应用。<br>
➁ 交互式用户环境中，为了避免"一个进程占用CPU从而拒绝为其它进程服务"，以及"某进程由于程序错误无限期阻塞其它进程"，必须使用抢占式调度算法。各种类型的服务器都属于此类系统(需要为多个不确定的客户提供服务)。<br>
➂ 在有实时限制的系统中，有时会出现不需要抢占式进程调度的情况：进程不会长时间占用CPU(通常很快执行完毕并进入阻塞态)。关于有实时限制的系统和交互式系统的差别，引用原文进行阐述：<br>
> __real-time systems__ run only programs that are intended to further the application at hand. <br>
__Interactive systems__ are general purpose and may run arbitrary programs that are not cooperative and even possibly malicious.

▶︎ __调度算法的目标(goals of scheduling algorithms)__ <br>
__一：__ __所有系统__ 执行调度算法时，都需要考虑的目标(all systems) <br>
➀ 公平(fairness): 给每个进程公平的CPU份额 <br>
➁ 策略强制执行(policy enforcement): 保证规定的策略被执行 <br>
➂ 平衡(balance): 保持系统的所有部分都处于"busy"状态 <br>

__二：__ __批处理系统场景下__ 应用调度算法，需要考虑的目标(batch systems) <br>
➀ 吞吐量(throught): 每小时最大作业数(越多越好) <br>
➁ 周转时间(turn-around time): 从提交到终止间的最小时间 <br>
➂ CPU利用率(CPU utilization): 保持CPU始终处于"busy"状态 <br>

__三：__ __交互式系统场景下__ 应用调度算法，需要考虑的目标(interactive systems) <br>
➀ 响应时间(response time): 快速响应请求(request) <br>
➁ 均衡性(proportionality): 满足用户的期望(expectations) <br>
 
__四：__ __实时系统场景下__ 应用调度算法，需要考虑的目标(real-time systems) <br>
➀ 在截止时间内完成进程调度(meeting deadlines): 避免丢失数据 <br>
➁ 可预测性(predictability): 在多媒体系统中避免音乐品质降低 <br>

##### 4.2 批处理系统中的调度算法(scheduling in batch systems)
▶︎ __先到先得(first come first served)__ <br>
先来先服务是一种非抢占式调度算法：所有进程根据它们❮请求CPU的顺序❯依次在❮就绪进程队列中❯进行处理。

➤ 其实现原理如下图所示：
<center><img src="/img/in-post/operating_system_img/os_2_37.pdf" width="100%"></center>

➤ 算法优点：易于理解并且便于在程序中运用。<br>

➤ 算法缺点：当多个竞争CPU的进程中，既包含CPU密集型进程，又包含I/O密集型进程时，则先到先得调度算法将导致I/O密集型进程的流程被拖慢数倍，影响了进程正确的运转效率。参考下面的例子理解： 

给定一个2进程竞争CPU资源的场景，一个进程为CPU密集型，另一个进程为I/O密集型，如下所示：
<center><img src="/img/in-post/operating_system_img/os_2_38.pdf" width="100%"></center>

绘制2个进程在CPU时序切换的示意图如下：
<center><img src="/img/in-post/operating_system_img/os_2_39.pdf" width="100%"></center>
可以看出，本来很快就可以完成的I/O密集型进程，由于先到先得调度算法，不断地被CPU密集型进程打断，最终需要近1000s才能完成。如果调度算法能够"帮助"I/O密集型进程每隔10ms抢占一次CPU，则I/O密集型算法大约只需10ms×1000=10s即可完成全部任务。

▶︎ __最短作业优先(shortest job first)__ <br>
最短作业优先是一种非抢占式调度算法，该算法适用于进程运行时间可以预知的情况：当输入队列中有若干个同等重要的作业处于就绪态时，调度程序优先选择执行完毕所需时间最短的进程优先执行。

➤ 最短作业优先调度算法的原理、特性：
<center><img src="/img/in-post/operating_system_img/os_2_40.pdf" width="90%"></center>

➤ 证明最短作业优先调度算法，在平均调度时间上为最优算法：<br>
不妨考虑4个进程作业的场景，设其运行时间分别为：a、b、c、d，则第一个作业需要时间a结束，第二个作业需要时间a+b，第三个作业需要时间a+b+c，第四个作业需要时间a+b+c+d才能结束。<br>
易知，上述4个进程完成作业的平均周转时间(trunaround time)为：__(4a+3b+2c+1d)/4__。因此，最短作业优先进行以此类推，最终得到最小的平均周转时间。<br>
但是，需要注意：最短作业优先的前提是：当前调度范围内的所有进程都处于 __同时可运行__ 的状态，否则最短作业优先在需要考虑 __进程等待时间__ 的前提下，不一定是最优解。

▶︎ __最短剩余时间优先(shortest remaining time next)__ <br>
最短作业优先是一种抢占式调度算法，是 __最短作业优先的抢占式版本__：当新进程作业到达时，调度程序将其整个执行时间同当前运行进程的剩余执行时间做比较，并选择剩余运行时间最短的那个进程执行(涉及进程间的抢占)。最后，同最短作业优先算法一致，它们需要掌握各个进程的运行时间信息。

##### 4.3 交互式系统中的调度算法(scheduling in interactive systems)
▶︎ __轮转调度算法(round-robin scheduling)__ <br>
轮转调度算法是一种古老、简单、公平、使用较广的算法：每个进程被分配一个时间段/时间片(quantum)，只允许进程在该时间片中运行；如果进程执行时间超过时间片，则剥夺其CPU资源并分配给另一个进程；如果进程在时间片结束前阻塞/结束，则CPU立即进行切换。

➤ 轮转调度算法的原理、特性：
<center><img src="/img/in-post/operating_system_img/os_2_41.pdf" width="100%"></center>

➤ 轮转调度算法中关于时间片的设置的权衡：<br>
<center><img src="/img/in-post/operating_system_img/os_2_42.pdf" width="80%"></center>
很容易发现，给定进程/上下文切换时间为1ms(用于切换寄存器、进程相关所有状态列表、内存映像、高速缓存...)，进程调度时间片从4ms变为100ms后，CPU的利用率上限显著地提升。

80% CPU利用率上限显然是对计算资源的极大的浪费，然而100ms时间片对应的99% CPU利用率上限方案同样存在无法回避的问题：
<center><img src="/img/in-post/operating_system_img/os_2_43.pdf" width="90%"></center>
因此，轮转调度算法的时间片设置必须权衡 __CPU利用率上限__ 以及 __进程轮转延迟__ 2方面考量。

除此以外，时间片设置的另外一个考虑因素是：如果设置的时间片比"进程平均CPU突发时间(CPU burst)"长，那么进程之间的抢占不会经常发生，另一方面，进程应当在时间片消耗完之前执行阻塞操作(perform blocking operation)，主动引发进程切换。

减少进程之间抢占是十分有益的：当进程的切换只发生于存在"实实在在"的运行逻辑需要时，将显著改善整体程序性能。推荐将时间片设置为20-50ms作为一个合理的折中。

▶︎ __优先级调度算法(priority scheduling)__ <br>




## Reference
> \<modern operating system 4th\> chapter2 <br>

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
> 12 问题脚注: ### ???problem😫problem???  大标题标识符：▶︎ 小标题标识符：➤ <br>
> 13 实现文章排版tab：`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;` <br>
> 14 项目序号 ⓪➀➁➂➃➄➅➆➇➈➉ ➊➋➌➍➎➏➐➑➒➓
