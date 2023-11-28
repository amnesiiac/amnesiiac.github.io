---
layout: post
title: "cpp design pattern - singleton"
subtitle: '[creational pattern] c++设计模式之单例模式'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-07-22 21:32
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
  - problem 
---
## 线程安全与单例模式
**[1] 什么是线程安全** <br> 在共享数据的多条线程并行执行的程序中，线程安全的代码会通过同步机制保证每个线程能够被正确地执行，不会出现数据污染的情况(多个线程同时操作同一块资源导致混乱)。

**[2] 如何保证线程安全** <br>
(1) 给共享的资源加把锁，保证每个资源变量每个时刻仅仅被一个线程占用。<br>
(2) 给予每个线程自己独有的资源，而不是多个线程共享进程中的资源(每个NBA球员给一个球，不就不用抢了么:-))。如threadlocal可以为每个线程维护一个私有本地变量。

**[3] 为什么需要单例模式** <br> 对于一个程序系统而言，有些时候确保某些类只有一个实例很重要。如：一个操作系统只能有一个系统时钟；打印机系统中可以有多个打印任务，但是只能有一个正在执行任务。那么，如何保证一个类只有一个对象实例并易于访问呢？答案就是使用单例模式(singleton)。

**[4] 单例模式的定义** <br> 单例模式确保某个类只有一个实例，且它自行实例化并向整个系统提供这个实例，这个类称之为单例类，该类提供实例的全局访问方法。要点有三：(1) 单例类只有一个实例；(2) 单例类必须自行创建该实例；(3) 单例类必须自行向整个系统提供该实例的访问方法。

**[5] 单例模式的分类** <br>
单例模式可以分为**懒汉式**和**饿汉式**，两者的区别在于创建单例的时间不同。<br>
**懒汉式**：在线程运行过程中，实例并不存在，只有当系统需要使用该实例时，才创建该实例(这种方式需要考虑线程安全)。<br>
**饿汉式** 线程启动时即创建实例，当系统需要使用该实例时，直接调用即可(本身线程安全，没有多线程操纵该实例的问题)。

**[6] 单例模式特点** <br>
(1) 构造函数和析构函数为private类型，为了禁止外部进行类型实例构造和析构。<br>
(2) 拷贝构造(copy ctor)和赋值运算符为private类型，禁止了外部的拷贝和赋值，确保了实例的唯一性。<br>
(3) 单例类含有获取实例的**静态**函数，可以全局访问。

**[7] strength & weakness** <br>
**strength** <br>
**1** 所创建的类型仅有一个实例(需要线程安全性保证)。**2** 对于该实例的访问只有一个全局接口。**3** 该实例仅在首次访问时进行初始化(只初始化一次)。<br>
**weakness** <br>
...


### 普通的懒汉式单例模式(线程不安全)
**[1] 懒汉式单例类定义**
```cpp
#include <iostream> // std::cout
#include <mutex> // std::mutex
#include <pthread.h> // pthread functions
class SingleInstance{
    private:
        // 将ctor dtor设置为私有 禁止外部构造析构
        SingleInstance();
        ~SingleInstance();
        // 将copy ctor和op=设置为私有 禁止复制和赋值
        SingleInstance(const SingleInstance&);
        const SingleInstance& operator=(const SingleInstance&);
    public:
        static SingleInstance* get_instance();// 静态的获取实例api
        static void delete_instance();// 释放单例资源 进程退出时调用
        void get_instance_address();// 打印单例地址
    private:
        static SingleInstance* ptr_single_instance;// 指向唯一单例的指针
};
// 静态常量成员必须类外初始化(特例:static const integral类型)
SingleInstance *SingleInstance::ptr_single_instance=nullptr;
SingleInstance* SingleInstance::get_instance(){// 申请资源
    if(ptr_single_instance==nullptr){
        // 不加锁的线程是不安全的 当线程并发时会创建多个实例
        // 使用new(std::nothrow)方式应对运行错误
        // 采用expression new申请资源+调用ctor创建对象
        ptr_single_instance = new(std::nothrow) SingleInstance;
    }
    return ptr_single_instance;
}
void SingleInstance::delete_instance(){// 释放资源
    if(ptr_single_instance){
        delete ptr_single_instance;
        ptr_single_instance=nullptr;
    }
}
void SingleInstance::print(){// print address
    std::cout<<"memory address: "<<this<<std::endl;
}
SingleInstance::SingleInstance(){// ctor def
    std::cout<<"ctor"<<std::endl;
}
SingleInstance::~SingleInstance(){// dtor def
    std::cout<<"dtor"<<std::endl;
}
```
**[2] 懒汉式多线程程序的具体实现(线程不安全，多个线程可能分别独占一个单例实例)**
```cpp
void printhello(void *thread_id_p){// 线程执行函数
    // 将创建的子线程设置为detached状态 -> 不关心其结束状态 OS在其结束时自动回收资源
    pthread_detach(pthread_self());
    // 对传入指针进行强制类型转化 void*->int*->int
    int thread_id = *((int*)thread_id_p);
    std::cout<<"thread id: ["<<thread_id<<"]"<<std::endl;
    // 创建单例模式单例 并打印其地址
    SingleInstance::get_instance->print();
    // terminate the calling thread 
    pthread_exit(nullptr);
}
const int NUM_THREADS=5;// 使用const int代替#define宏
int main(){
  pthread_t threads[NUM_THREADS]={0};// 创建thread_t类型数组 
  int indexes[NUM_THREADS]={0};// 创建index数组
  int ret=0; int i=0;
  std::cout<<"main(): start"<<std::endl;
  for(int i=0; i<NUM_THREADS; ++i){// 依次创建5 concurrency thread
    std::cout<<"main(): create threads: ["<<i<<"]"<<std::endl;
    indexes[i]=i;// 初始化index数组 -> 在pthread_create中转化成void*进行使用
    ret = pthread_create(&threads[i],nullptr,printhello,(void*)(indexes[i]));
    if(ret){// 线程创建成功返回ret=nullptr
      std::cout<<"error: cannot create threads"<<std::endl;
      exit(-1);
    }
  }
  // 理想情况(不产生线程安全问题) 普通的懒汉单例多线程模式只会创建1个instance
  // 实际上(发生线程安全问题) 普通的懒汉式单例多线程模式创建了3个instance 
  SingleInstance::delete_instance();// 手动释放申请的资源 释放的是哪一个进程实例?
  std::cout<<"main(): end"<<std::endl;
  return 0;
}
```
**[3] 普通懒汉式程序运行结果及分析(详见程序输出注释)**
```txt
[root@]# g++ SingleInstance.cpp -o SingleInstance -lpthread -std=c++0x
[root@]# ./SingleInstance

main(): start 
main(): createthread: [0]      // 依次创建线程 1-5
main(): createthread: [1]
main(): createthread: [2]
thread id: [1]                 // 线程是并行的 每个线程内运行的程序顺序互相独立
thread id: [0]
main(): createthread: [3]
thread id: [2]
ctor                           // (a) 调用singleton构造函数 
memory address: 0x7f9ef00008c0 // no.1 独立的单例地址
main(): createthread: [4]
ctor                           // (b) 调用singleton构造函数 
memory address: 0x7f9ee80008c0 // no.2 独立的单例地址
thread id: [3]
memory address: 0x7f9ee80008c0 // no.2 和单例地址2共享同一实例
ctor                           // (c) 调用singleton构造函数
memory address: 0x7f9eec0008c0 // no.3 独立的单例地址
dtor                           // 调用dtor随机释放一个进程产生的instance(发生资源泄漏)
thread id:[4]
memory address: 0x7f9eec0008c0 // no.3 和单例地址3共享同一实例
main(): end

```
**[4] 为什么普通的懒汉式的单例模式是进程不安全的** <br>
懒汉式的单例实例是第一次使用时即刻创建的，创建的基本逻辑是：首先判断是否已经有单例，如果没有则创建单例，否则不创建。但是，由于管理单例的指针没有加锁，当多个线程同一时刻使用该单例模式时，同时判断是否有单例，则可能导致判断结果均为"尚未创建"，因此会重复创建单例的实例，从而是线程不安全的。

**[5] 关于得到的结果中的补充分析** <br>
(1) 根据程序输出结果：调用了3次ctor，1次dtor，从对象地址上可以看出，实际创建了3个singleton而不是程序设计预想中的1个，有两个进程和其他进程共享singleton，因此不是线程安全的。另外，上述程序在执行的过程中会发生资源泄漏的问题：申请了3个单例对象的资源，最终只释放了1个。<br>
通过**RAII**技术可以一定程度上保证资源不发生泄漏(但是可能和程序设计预期功能产生冲突)，但是线程仍然是不安全的(普通的懒汉式程序无法在多个线程自由组织运行时保证只创建一个单例对象)。<br>
(2) 另外，在创建线程之前，通过pthread_detach()函数将创建的线程设置成detached，从而在线程完成任务时，由操作系释放放其占用的资源；因此，对于本例中没有调用dtor释放资源的其他线程，交由OS进对其进行资源释放。<br>
(3) 根据[pthread_exit函数文档](https://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_exit.html): "Thread termination does not release any application visible process resources, including, but not limited to, mutexes and file descriptors, nor does it perform any process-level cleanup actions"，易知：printhello函数只进行资源申请，但是进程退出不保证资源释放。<br>

**[6] 关于linux中的pthread的两种状态：joinable & detached** <br>
(1) linux中的pthread有两种状态，一种是joinable状态，另一种是detached状态。如果线程被创建or设置为joinable状态，当线程函数自行结束返回退出or调用pthread_exit()函数手动退出时，不会释放线程占用的堆栈以及线程描述符；但是它可以被其他线程收回其资源并杀死，或手动调用pthread_join()执行后将其资源释放。如果线程被创建or设置为detached状态，其占用的资源在线程函数退出or调用pthread_exit()时被自动释放。<br>
(2) 关于线程中的其他内容，可以参考博客中对\<c++ concurrency in action\>教材的相关知识的整理，以及。

**[7] 关于使用了POSIX pthread线程库的程序编译选项的问题** <br>
(1) 使用了linux下的多线程库pthread，则需要使用lpthread或者pthread编译选项。<br>
(2) lpthread只对链接器起作用，pthread对预处理器和链接器起作用(推荐使用pthread)。<br>
(3) 使用lpthread时，需要将lpthread放在源文件之后否则可能报错(pthread无此限制)。<br>

### 加锁的懒汉式单例模式(线程安全)
**[1] 加锁的懒汉式单例类定义**
```cpp
class SingleInstance{
    private:
        SingleInstance();// ctor
        ~SingleInstance();// dtor
        SingleInstance(const SingleInstance&);// copy ctor
        const SingleInstance& opereator=(const SingleInstance&);// op=
    public:
        static SingleInstance* get_instance();// 获取单例
        static void delete_instance();// 删除单例
        void print();// 打印单例地址信息
    private:
        static SingleInstance* ptr_single_instance;// 单例
        static std::mutex single_instance_mutex;// 互斥锁
};
// init static members
SingleInstance* SingleInstance::ptr_single_instance = nullptr;
std::mutex SingleInstance::single_instance_mutex;// ?????? 这是什么初始化方式
SingleInstance* SingleInstance::get_instance(){
    // 为单例指针(资源)构建'双检锁'(应用了两个if) 
    if(ptr_single_instance == nullptr){// 没有单例 不加锁
        std::unique_lock<std::mutex> lock(single_instance_mutex);// 上锁
        if(ptr_single_instance == nullptr){
            ptr_single_instance = new(std::nothrow) SingleInstance;
        }
    }
    return ptr_single_instance;
}
void SingleInstance::delete_instance(){
    // ??????????? 为什么这个销毁资源之前还要上锁
    std::unique_lock<std::mutex> lock(ptr_single_instance);// 上锁
    if(ptr_single_instance){
        delete ptr_single_instance;
        ptr_single_instance = nullptr;
    }
}
void SingleInstance::print(){// 打印当前单例的地址
    std::cout<<"memory address: "<<this<<std::endl;
}
SingleInstance::SingleInstance(){// ctor
    std::cout<<"ctor"<<std::endl;
}
SingleInstance::~SingleInstance(){// dtor
    std::cout<<"dtor"<<std::endl;
}
```
**[2] 懒汉式多线程程序的具体实现(实现方式同普通懒汉式一致)** <br> 普通懒汉式线程不安全，但是通过在操纵的单例类中给资源加锁的方式使其安全。

**[3] 加锁懒汉式程序运行结果及分析(详见程序输出注释)**
```txt
[root@]# g++ SingleInstance.cpp -o SingleInstance -lpthread -std=c++0x
[root@]# ./SingleInstance 

main(): start 
main(): create thread: [0]     // 依次创建多个线程
main(): create thread: [1]
main(): create thread: [2]
thread id: [0]
main(): create thread: [3]
thread id: [1]
ctor                           // 调用get_instance构建唯一的单例资源    
memory address: 0x7f28b00008c0 // no.1 static单例对象地址 
memory address: 0x7f28b00008c0 // no.2 static单例对象地址
thread id: [2]
memory address: 0x7f28b00008c0 // no.3 static单例对象地址
main(): create thread: [4]
thread id: [3]
memory address: 0x7f28b00008c0 // no.4 单例对象地址 
dtor                           // 调用delete_instance释放唯一的单例资源
main(): end
```
**[4] 加了互斥锁(mutex)的懒汉单例模式为什么是线程安全的(结合上述实验输出进行分析)** <br>
(1) 多线程程序中只会调用1次ctor创建1个单例：由于在double-checked mutexed singleton中，给资源管理指针上了锁，使其不能被多个线程共享，因此同一时刻只有一个thread进行资源分配操作，第一个thread完成资源分配之后，通过双检锁自动屏蔽后续thread的资源分配请求。<br>
(2) 只需调用一次dtor销毁了一个单例：double-checked mutexed singleton只有一个实例。

**[5] 尽管将普通的懒汉式单例程序加锁是线程安全的，但是加锁会损失程序的性能** 

### 局部静态变量的懒汉式单例模式(c++11线程安全)
**[1] 局部静态变量的懒汉式单例类定义**
```cpp
class SingleInstance{
    private:
        SingleInstance();
        ~SingleInstance();
        SingleInstance(const SingleInstance&);
        const SingleInstance& operator=(const SingleInstance&);
    public:
        static SingleInstance& get_instance(); 
        void print();
};
SingleInstance& SingleInstance::get_instance(){// 获取单例
    static SingleInstance local_static_instance;// 调用ctor创建局部静态变量单例
    return local_static_instance;
}
SingleInstance::SingleInstance(){// ctor
    std::cout<<"ctor"<<std::endl;
}
SingleInstance::~SingleInstance{// dtor
    std::cout<<"dtor"<<std::endl;
}
```
**[2] 局部静态变量的懒汉式单例类的具体实现(实现方式同普通懒汉式一致)** <br> 

**[3] 局部静态变量的懒汉式程序运行结果及分析(详见程序输出注释)**
```txt
[root@]# g++ SingleInstance.cpp -o SingleInstance -lpthread -std=c++0x
[root@]# ./SingleInstance 

main(): start 
main(): create thread: [0]  // main函数依次创建0-4号线程
main(): create thread: [1]
thread id: [0]              // 0号线程执行
ctor                        // 调用get_instance创建局部静态单例对象
memory address: 0x6016e8
thread id: [1]              // 1号线程执行
memory address: 0x6016e8
main(): create thread: [2]
main(): create thread: [3]
main(): create thread: [4]
thread id: [3]              // 3号线程执行
memory address: 0x6016e8
thread id: [2]              // 2号线程执行
memory address: 0x6016e8
main(): end 
dtor                        // 局部变量生命期结束 系统调用dtor销毁局部static变量 
```
**[4] 关于局部静态变量懒汉式单例模式程序的补充说明** <br>
程序运行结束，局部静态单例对象被销毁，自动调用dtor，因此无需为单例类实现delete_instance。

### 普通的饿汉式单例模式(本身线程安全)
**[1] 普通的饿汉式单例类定义**
```cpp
class SingleInstance{
    private:
    // 将ctor、dtor、copy ctor、op=设为私有 禁止外部创建、销毁、复制、赋值单例
        SingleInstance();
        ~SingleInstance();
        SingleInstance(const SingleInstance&);
        const SingleInstance& operator=(const SingleInstance&);
    public:
        static SingleInstance* get_instance();// 获取单例
        static void delete_instance();// 销毁单例
        void print();
    private:
        static SingleInstance *ptr_single_instance;// 单例资源管理指针 
};
// 饿汉式: 在类定义第一次运行即刻调用dtor创建单例 -> 线程安全
// 使用new(std::nothrow)进行单例分配 分配失败返回nullptr指针
SingleInstance 
    *SingleInstance::ptr_single_instance = new(std::nothrow) SingleInstance;
SingleInstance* SingleInstance::get_instance(){// 返回单例
    return ptr_single_instance;
}
void SingleInstance::delete_instance(){// 调用dtor销毁单例 释放空间
    if(ptr_single_instance){
        delete ptr_single_instance;
        ptr_single_instance = nullptr;
    }
}
void print(){
    std::cout<<"memory address: "<<this<<std::endl;
}
SingleInstance::SingleInstance(){// ctor
    std::cout<<"ctor"<<std::endl;
}
SingleInstance::~SingleInstance(){// dtor
    std::cout<<"dtor"<<std::endl;
}
```
**[2] 普通的饿汉式单例类的具体实现** <br>
**[3] 普通的饿汉式单例模式程序运行结果及分析(详见程序输出注释)**
```txt
[root@]# g++ SingleInstance.cpp -o SingleInstance -lpthread -std=c++0x
[root@]# ./SingleInstance 

ctor
main(): start 
main(): create thread: [0]  // main函数依次创建0-5线程
main(): create thread: [1]
thread id: [0]              // 正执行0号线程
memory address: 0xf80010    // 线程获取单例 并打印单例地址
main(): create thread: [2]
thread id: [1]
memory address: 0xf80010
main(): create thread: [3]
thread id: [2]
memory address: 0xf80010
main(): create thread: [4]
thread id: [3]
memory address: 0xf80010
dtor                        // ->delete_instance->dtor
main(): end
```
**[4] 关于普通的饿汉式单例模式程序的补充说明** <br>
饿汉式不同与懒汉式，懒汉式只有使用单例时，才创建单例对象，而饿汉式在第一次使用单例类定义即刻创建单例对象。

**[5] 懒汉式和饿汉式两种单例模式的比较** <br>
(1) 懒汉式是**空间换取时间**，适用于访问量较小的场景(线程较少的场景)。对于懒汉式涉及的3种基本方法，推荐使用局部静态变量的懒汉式单例模式，代码量少，相比加锁的懒汉式单例模式效率较高。<br>
(2) 饿汉式是**空间换取时间**，适用于访问量较大的场景(线程较多的场景)。

### ??????problem😫problem??????
关于加锁的懒汉式单例模式的mutex使用方法？关于mutex的基本机制。

## Reference
> cpp design pattern(GOF) chapter 1<br>
> https://zhuanlan.zhihu.com/p/83539039 <br>
> https://refactoringguru.cn/design-patterns/singleton <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式: <br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色
