---
layout: post
title: "cpp - smart pointer"
subtitle: '智能指针相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-07 22:25
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---
### 三种内存空间简述
static静态内存用于存储局部static对象、类的static成员以及定义在任何函数之外的变量。栈内存用来保存定义在函数内部的非static对象。分配在static静态内存、栈内存中的对象通过编译器对其生命期进行管理(动态的构建或者销毁)。<br>
除了上述两种内存空间，每个应用程序还有一个内存池，称之为自由存储空间或者堆；程序将所有的动态对象存储在这个内存池中，动态对象的生命期由程序管理。


### 智能指针简述
在[这篇博客](/cpp/2021/05/07/post-cpp-new-delete/)中，介绍了有关动态内存分配销毁的相关知识(new & delete)。使用new & delete对于对象的生命期的管理较为底层，并且在对象释放过程中，容易引发内存泄漏、空悬指针等一些棘手的问题。<br>
为了解决上述问题，c++提供了一组智能指针，用于完成动态对象的管理(自动销毁所指向的对象)。智能指针包括**shared_ptr**、**unique_ptr**、**weak_ptr**。

**[智能指针和异常]** <br>
程序块结束有两种方式，一种正常结束，一种执行出现异常而结束，两种方式都会从异常产生处直接跳转到快结束处，并且销毁程序块中定义的局部对象(包括各种指针)。使用智能指针可以在两种情况下，都能够正确的管理相应的内存；然而，使用new&delete操作符的方式管理的对象(内存)可能在出现异常并且没有异常捕获处理机制下，不能够正常被释放。可以结合下面的程序进行分析：
```c++
// smart pointer 
void func(){
    shared_ptr<int> s(new int(1024));
    // 这段代码出现异常 并且没有相应的异常捕获机制进行处理 -> 函数块被强制结束 
    // 函数块强制结束 块中创建的指针s被销毁 如果s.use_count()==0 则s指向的内存被自动释放
}
// new & delete
void func(){
    int *ptr = new int(1024);
    // 这段代码出现异常 并且没有相应的异常捕获机制进行处理 -> 不执行delete 函数块强制结束
    // 函数块强制结束 ptr被销毁 delete ptr被跳过 因此指向内存泄漏
    delete ptr;
}
```
智能指针很擅长处理那些：在资源被正确的释放之前，容易产生未知的异常(没有捕获或者不易捕获)的情况；以及很容的处理**dumb class**资源回收的相关问题(定义了构造函数，但是没有很好的定义析构函数的类)：对象创建并且使用完成后，占用的资源没有得到正确的释放。

**[智能指针使用的注意事项]** <br>
**[1]** 不使用相同的内置指针初始化或者reset不同的智能指针。使用相同的内置指针初始化或reset不同的智能指针导致智能指针之间是独立的，因此不共享计数器，破坏了智能指针对于对象(内存)的管理能力。<br>
**[2]** 不使用get()初始化或者重定位另外一个智能指针。原因和**[1]**一致，指向相同对象(资源)的不同智能指针破坏了智能指针的管理资源能力。<br>
**[3]** 不通过delete对于get()函数返回的内置指针。简而言之：不要通过new&delete方式对于智能指针指向的对象(资源)进行管理，容易产生管理上的冲突。<br>
**[4]** 如果程序使用了get()返回的内置指针，则需要注意指针指向的资源什么时候被对应智能指针自动释放(注意内置指针变成空悬指针的时机)。<br>
**[5]** 如果你使用的智能指针管理的资源不是new分配的内存，则应该给他传递一个删除器。

**[智能指针通用的操作]** <br>
**[1]** 通过`解引用`的方式获得指向的对象：
```c++
shared_ptr<string> ptr1(new string("hello"));
unique_ptr<string> ptr1(new string("hello"));
if(ptr1 && *ptr1){// 检测指针和指向的对象是否为空
    *ptr1="new string";
}
```
**[2]** 通过`->`操作符访问成员对象：
```c++
// shared_ptr
shared_ptr<myclass> class_ptr=make_shared<myclass>();// 执行myclass默认初始化
class_ptr->myclass_func();// 通过智能指针调用指向对象的方法
// unique_ptr
unique_ptr<myclass> class_ptr(new myclass());// 只能用直接初始化
class_ptr->myclass_func();
```
**[3]** 通过`.get()`函数返回智能指针保存的底层指针。这种用法有时是非常危险的：动态指针被自动销毁后，继续使用get函数返回的指针是非常危险的(变成空悬指针)。
```c++
// shared_ptr
shared_ptr<string> ptr=make_shared<string>(10, "9");// 调用string构造创建对象
string *normal_ptr = ptr.get();// 通过get函数返回相应底层指针
// unique_ptr
unique_ptr<string> ptr(new string("hello"));// 不支持make_shared只能用直接初始化
string *normal_ptr = ptr.get();
```
**[4]** 通过`swap`函数交换两个智能指针。在[cpp_doc](http://www.cplusplus.com/reference/utility/swap/?kw=swap)可以找到完成swap函数的条件：用于交换的类型必须是`move constructible`以及`move assignable`的。这是因为swap函数实际上调用了T模版类型的**右值引用构造函数**以及**右值引用赋值函数**。如果两个函数被定义成**=delete**，则swap失败。
```c++
// shared_ptr
shared_ptr<string> ptr1=make_shared<string>(10, "9");
shared_ptr<string> ptr2=make_shared<string>(4, "8");
// unique_ptr
unique_ptr<string> ptr1(new string(10, "9"));
unique_ptr<string> ptr2(new string(4, "8"));
// swap
swap(ptr1, ptr2);// 交换两个相同模版类型的指针
ptr1.swap(ptr2);// 一个等价的方式 - 上面的方法调用这个实现指针交换
```
**[5]** 通过`new`运算符初始化智能指针。使用new初始化智能指针必须使用直接初始化的形式。
```c++
// 间接初始化 -> 错误 -> 隐式地将内置类型指针转化成智能指针 (c++没有定义这种转化)
shared_ptr<int> ptr_int = new int(0);// no.1
unique_ptr<int> ptr_int = new int(0);// no.1
int *ptr = new int(0); shared_ptr<int> ptr_int = *ptr;// no.2
int *ptr = new int(0); unique_ptr<int> ptr_int = *ptr;// no.2
// 直接初始化 -> 正确 -> 不涉及隐式转化
shared_ptr<int> ptr_int(new int(0));
unique_ptr<int> ptr_int(new int(0));
```

### shared_ptr
shared_ptr同标准库容器vector类似，都是通过模版实现的：
```c++
#include <memory>
#include <string>
#include <list>
shared_ptr<string> ptr1;// 指向一个string
shared_ptr<list<int>> ptr2;// 指向存储了int类型元素的链表
```

**[shared_ptr独占的操作]** <br>
**[1]** 通过`make_shared`操作符对于shared_ptr进行初始化:
```c++
shared_ptr<string> ptr=make_shared<string>("hello");
```
**[2]** 通过`p(q)`的方式创建shared_ptr的拷贝，并且将被拷贝的指针q指针计数器+1。需要注意的是：q的指针类型须相同或者能够隐式的转化成p指针定义的类型。
```c++
// no.1 
shared_ptr<string> p(q);
// no.2 通过unique_ptr构建shared_ptr - 将unique_ptr置空
unique_ptr<string> u(new string("hello"));
shared_ptr<string> p(u);
// no.3 通过内置指针类型构建shared_ptr: 一旦构建完成 不要再通过内置指针管理这块内存
string *uu = new string("hello");
shared_ptr<string> p(uu, new_delete);// 对象释放时将调用new_delete
// no.4 通过shared_ptr构建shared_ptr 
shared_ptr<string> uuu(new string("hello"));
shared_ptr<string> p(uuu, new_delete);// 对象释放时将调用new_delete
```
**[3]** 两个shared_ptr之间的赋值`=`操作。要求是两个shared_ptr的T模版类型能够相互转化。
```c++
shared_ptr<char> ptr1=make_shared<char>('c');
shared_ptr<int> ptr2=make_shared<int>(90);
ptr2=ptr1;// ptr2的指针计数器-1 ptr1的指针计数器+1 ptr2的计数器为0则释放原内存
```
**[4]** 通过`.unique()`函数，判断shared_ptr指针计数器是否为1:
```c++
ptr1.unique();// 计数器=1 返回ture 否则返回false
```
**[5]** 通过`.use_count()`函数，返回shared_ptr指针共享对象的指针数量。运行可能较慢，主要用于程序调试。
```c++
ptr1.use_count();// 查看ptr1指向的对象被多少指针管理
```
**[6]** 通过`reset()`函数，shared_ptr重定向，或者释放指向的对象并置shared_ptr为空。
```c++
shared_ptr<T> ptr(new T);
ptr.reset();// 如果ptr.use_count()=1 则销毁对象释放内存 置ptr为空
ptr.reset(q);// 将ptr重新向到内置指针q() U-T之间可以隐式转化
ptr.reset(q, d);// 基本功能同上 但是对象的销毁通过d进行 而不是默认的delete
// 例子:
T *a;
shared_ptr<T> ptr(a);// *ptr=*a
shared_ptr<T> ptr(new T());// *ptr=调用T类型默认构造函数创建的对象
```

**[shared_ptr通过指针计数器进行对象的管理]** <br>
指针计数器记录了当前程序运行时刻有多少个指针**管理**同一个对象。在自由存储区创建的对象没有名字，只能通过指针进行管理，当指针计数器=0，表明没有指针继续管理这个对象(对象生命期结束)，即调用相应类型的析构函数将这个对象自动销毁，相应内存被自动释放。

**[shared_ptr自身的管理]** <br>
智能指针能够实现所管理的对象(内存空间)自动的销毁释放，但是，其本身的生命期如果长于对象生命期，则依然会出现试图调用该指针的情况，使用dangling的指针是未定义的，因此，建议将所有shared_ptr管理在一个容器中，在执行指针管理的过程中按照**指针计数器**进行排序，并且将计数器=0的指针销毁erase掉。

**[不要将内置指针和shared_ptr混用]** <br>
混用内置指针和shared_ptr往往是危险的，如果一个对象同时被shared_ptr和内置指针进行管理，当shared_ptr指针计数器为0时，指向的对象被自动销毁，内存被释放，再使用内置指针操作这块内存的结果是为定义的。
```c++
int *p = new int(1024);
process(shared_ptr<int> (p));// 创建了临时shared_ptr 函数执行完 指针计数=0
int a = *p;// 未定义的操作 p指向的内存已经被释放 -> dangerous!
```
**[不要使用get函数初始化另外的shared_ptr指针或为其赋值]**
```c++
shared_ptr<int> p(new int(42));
int *q=p.get();// 将p管理的对象(内存)供内置指针q管理 注意p销毁后 不要使用q
{shared_ptr<int> pp(q);}// 新程序块中 使用内置指针初始化智能指针 块结束 pp释放内存
int wrong=*p;// 未定义的操作 p指向的内存已经被释放 销毁p会引发内存二次delete!
```
上述代码中，两个智能指针**p**和**pp**之间是独立的，他们分别拥有自己的指针计数器，智能指针共享对象的协同性被打破。关于get函数的正确使用方式：get函数应该用于通过一个智能指针向使用传统内置指针的代码段(只使用内置指针)中传递一个初始值。总结：不要混用内置指针和智能指针，使用内置指针的代码段只使用内置指针，使用智能指针的部分只使用智能指针。两者之间可以通过get、直接初始化的方式进行链接。

**[关于指针计数器]** <br>
指针计数器用于对特定指针指向特定内存的数量进行计数。因此，指针计数器属于指针，表明了当前：包含这个指针及其衍生的shared_ptr(指向相同的内存)的总数量。需要注意的是：可以存在多个**独立**的shared_ptr指向同一对象(内存)，每个独立的shared_ptr有自己的计数器(如前一部分代码所示)。尽量不要这么做，这种情况往往使得各个指针难于管理。

**[什么情况下使用shared_ptr]** <br>
工1程中需要使用动态的内存管理主要由于下面的三种情况：<br>
**[1]** 程序本身不确定需要构建、使用多少对象。<br>
一个非常典型的例子是：容器类使用动态内存管理机制进行对象管理，如vector类，在创建一个vector容器时，不能确定在这个容器中需要容纳多少元素。<br>

**[2]** 程序不知道所构建的对象的准确的类型。<br>
这种情况下使用动态内存分配的原因和上面类似：程序不能确定构建对象的准确类型，即不能判断需要预先分配多少空间(静态的)，因此只能使用动态的内存管理机制。<br>

**[3]** 程序需要在多个对象之间共享数据。<br>
很多情况下，很多工程实践中，多个对象都共享了相同的状态，即很多对象都拥有在内存中同一份状态的实体。此时，当一个对象被销毁时，我们不能简单地销毁这份共享的状态实体，如果有其他的对象正在使用这份实体，则它应该保留。在上述的应用背景下，应当使用带有计数器的动态内存管理机制。

### unique_ptr
unique_ptr**拥有**其指向的对象，而不是同shared_ptr一样可以和其他的shared_ptr共享这个对象。特定的时刻，只能有一个unique_ptr指向一个对象。

**[unique_ptr独占的操作]** <br>
**[1]** 通过一个特定类型`D`的对象来释放其指针，在unique_ptr创建时即指定删除器的类型。
```c++
unique_ptr<T, D> u;
```
**[2]** 通过`D`类型对象`d`来代替原生delete，在unique_ptr创建时即指定删除器类型及实体。
```c++
unique_ptr<T, D> u(d);
```
**[3]** 使用`nullptr`显式销毁指针所指对象，将指针置为空。
```c++
unique_ptr<T> u(new T());
u=nullptr;
```
**[4]** 使用`release`函数放弃指针对于对象的控制权，并且置指针为空，返回这个空指针。release操作不会销毁原指向的对象。
```c++
unique_ptr<T> u(new T());
u.release();// 放弃u对其对象的控制权 并将u置为空
```
**[5]** 使用`reset`函数重定向指针。reset操作会将原指向的对象销毁。
```c++
unique_ptr<T> u(new T());
u.reset();// 销毁u指向的对象 并释放相应内存 
u.reset(ptr);// 将u重定向到内置指针ptr指向的对象上 
u.reset(nullptr);// 将u重定向为空指针
```
**[6]** 使用`reset`或者`release`函数将对象的所有权从一个unique_ptr转移到另外一个unique_ptr。空的unique_ptr(不指向任何对象实体的)可以用来初始化其他unique_ptr指针。
```c++
unique_ptr<T> u(new T());
unique_ptr<T> uu(u.release());// 放弃u对其对象的控制权 u置为空 返回u用于初始化uu
unique_ptr<T> uuu(new T());
uuu.reset(u.release());// 释放uuu控制的对象 放弃u对其对象控制权 u置为空 重定向uuu 
```

**[传递unique_ptr作为参数or将unique_ptr作为返回值]**
unique_ptr一般不支持拷贝or赋值操作，是为了保证unique_ptr对于其管理的对象的**独占**管理权。但是，在特殊的情况下，我们可以拷贝or赋值一个即将被销毁的unique_ptr指针。
```c++
unique_ptr<int> clone(int num){
    return unique_ptr<int>(new int(num));// unique_ptr作为函数返回值
}
```
上述机制是通过移动构造函数完成的，即将一个unique_ptr指针对象的资源转移到另外一个unique_ptr指针对象中。

**[向unique_ptr传递删除器]**
unique_ptr可以在创建时或者调用reset函数时，显式的提供删除器类型以及删除器对象。unique_ptr管理删除器的方式和shared_ptr不同(参考c++primer 5th 16.16)。
```c++
// 创建unique_ptr对象时 显式的定义 删除器对象及其类型
unique_ptr<obj_type, del_type>(new obj_type obj, del);
```

### weak_ptr
weak_ptr是一种不控制所指对象生命期的智能指针，它指向一个由shared_ptr管理的对象，但是weak_ptr不参与shared_ptr指针的计数。weak_ptr所有的操作基本都和shared_ptr相关，可以看成weak_ptr是shared_ptr的一种辅助指针，不能对于和shared_ptr共享的对象进行销毁创建等操作，但是可以安全的(不涉及对象操作)对于shared_ptr的状态进行检测。

**[weak_ptr的使用方法]**
```c++
weak_ptr<T> w;// 初始化一个空的weak_ptr
weak_ptr<T> w(s);// 使用shared_ptr s初始化weak_ptr w
w = s; w1 = w2;// weak_ptr w和shared_ptr s共享对象; weak_ptr1和weak_ptr2共享对象
w.reset();// 将w置空 不对指向的对象进行销毁! -> 弱指针
w.use_count();// 返回对象上的shared_ptr个数
w.expired();// use_count==0? true:false
w.lock();// expired()==true? 返回空的shared_ptr : 返回指向w的对象的shared_ptr
```
注意，w.lock()函数只是返回一个指向w对象的shared_ptr，但是不能增加相应的shared_ptr的计数。

## Reference
> 《c++ primer》5th chapter 12

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
