---
layout: post
title: "cpp - new & delete"
subtitle: 'new & delete相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-07 17:35
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---
### new & delete的基本用法整理
**[x] new(std::nothrow)和new的区别** 在于两者对于运行错误处理方式不同。
```c++
#include <iostream>
#include <new>
int main(){
    try{// no.1 普通new资源分配失败直接抛出bad_alloc类异常
        while(true){
            new int[100000000ul];// throwing overload
        }
    }catch(const std::bad_alloc& e){
        std::cout<<e.what()<<'\n';
    }
    while(true){// no.2 nothrow overload将返回指针置为nullptr
        int* p=new(std::nothrow) int[100000000ul];// non-throwing overload
        if(p==nullptr){
            std::cout<<"Allocation returned nullptr"<<std::endl;
            break;
        }
    }
}
```



### 单个对象的动态分配和释放
**[new]**<br>
主要由\[new关键字\]+\[类型指示符\]构成，类型指示符可以是内置类型也可以是类类型：
```c++
new int;// 分配了一个内置类型的int对象
new myclass;// 分配了一个自定义类类型对象
```
new表达式并没有直接返回int或者myclass类的对象，而是间接返回了指向该类对象的指针。对这个对象的操作都需通过这个指针来完成。
```c++
int *ptr_int = new int;// 通过ptr_int操作new分配的内存空间
myclass *ptr_myclass = new myclass;// 通过ptr_myclass操作new分配的内存空间
```
new关键字分配的对象都是位于**空闲储存区**：`1` 在空闲储存区分配的对象没有名字(无名对象)，对象操作通过指针完成。`2` 在空闲存储区分配的内存是未经初始化的，处于**位随机模式**，内存中的值是上次使用时留下的数值。可以通过如下的方式对于new分配的空闲存储区进行初始化：
```c++
int *ptr_int = new int;// 执行默认初始化 对于内置类型 对象的数值是未定义的
int *ptr_int = new int();// 执行值初始化 对于内置类型 有良好定义的数值
int *ptr_int = new int(0);// 通过使用括号initializer给予空闲存储区一个初始值
myclass *ptr_myclass = new myclass;// 默认初始化 调用默认构造
myclass *ptr_myclass = new myclass();// 值初始化 调用默认构造 和上面等价的
myclass *ptr_myclass = new myclass(xxx);// 调用myclass类相关构造函数进行初始化
// 类似new的一种常规创建方式
int a=0; int *ptr_a = &a;// ptr_a的作用和ptr_int类似 但是存储在程序自由存储区中
```
需要补充说明的是：空闲储存区的资源是有限的，当空闲储存区的资源被耗尽时，使用new表达式会因为申请不到相应的内存而抛出**bad_alloc**异常。

**[空闲存储区or自由存储区]** <br>
**空间申请流程:** 操作系统有一个记录空闲内存地址的链表，当系统收到程序的申请时，会遍历该链表，寻找第一个空间大于所申请空间的堆结点，然后将该结点从空闲结点链表中删除，并将该结点的空间分配给程序。<br>
**记录分配的大小:** 对于大多数系统，会在这块内存空间中的首地址处记录本次分配的大小，这样，代码中的delete语句才能正确的释放本内存空间。<br>
**多分配heap空间的处理:** 由于找到的堆结点的大小不一定正好等于申请的大小，系统会自动的将多余的那部分重新放入空闲链表中。

**[heap和自由存储区之间的关系]** <br>
自由存储区是由new/delete来管理的，heap是通过malloc/free来管理的。<br>
注意的是，heap和自由存储区(freestore)可能是位于不同的物理内存区域，并且它们可能被不同的底层内存管理器控制。从技术角度来说，freestore是一个抽象的术语，它表示那些用来动态分配内存的未占用空间。而heap是一个具体形象的数据模型概念，它通过c++编译器去实现扩充了freestore。

**[delete]** <br>
delete用于释放new申请分配的空闲存储区内存，结束指针创建的相应类型的对象的生命期。
```c++
delete ptr_int;// 释放ptr_int指向的空间 结束ptr_int的生命期
delete ptr_myclass;// 释放ptr_myclass指向的空间 结束ptr_myclass生命期
```
注意，delete只能用于释放new创建的位于空闲存储区的对象，使用delete释放位于空闲存储区以外的对象，将会导致程序运行期间未定义的行为。常见的和delete相关的错误：<br>
`1` 调用delete失败，导致空闲存储区的内存没有正确释放，称之为内存泄漏(memory leak)。<br>
`2` 对于同一块空闲存储区应用了至少2次的delete。这种情况通常都是由于释放了多个指向同一块动态分配空间的指针导致的，多个指针指向相同的对象。<br>
`3` 在对象被delete释放后，仍然通过指针对其进行读写。这种情况大都是由于释放了对象，但没有将对应的指针置0导致的。

**[对象的生命期和指针的生命期]** <br>
注意，new表达式通过调用系统的库函数new()创建了一个`无名对象`，并且将初始化的指针指向该对象(ptr_int)，ptr_int的生命期和这个`无名对象`的生命期无关，当这个对象使用delete()销毁时，对象的生命期结束，但是并不影响ptr_int的生命期。如果ptr_int指针生命期长于它指向的对象，称之为空悬指针(dangling pointer)。为了避免这个指针后续被操作导致未知的错误，常常在销毁其指向的对象后，将ptr_int置为0或者nullptr，表明其不指向任何对象。

### 对象数组的动态分配和释放
new表达式也可以动态的在自由存储区分配一个数组：
```c++
int *ptr1 = new int(1024);// 分配一个int型对象 初始化为1024
int *ptr2 = new int[1024];// 分配一个具有1024个原始的数组 未被初始化
int (*ptr3)[1024] = new int[4][1024];// 分配一个4行1024列的数组
```
一般的，new表达式分配的数组不能直接进行初始化，自定义初始化值必须通过**for循环**来完成。<br>
另外，在new初始化数组的过程中，只有第一维可以指定一个运行时的动态值，其他维度需要在编译之前给出定值:
```c++
char *ptr1 = new char[get_dim()+1];// 第一维指定运行动态值
char (*ptr2)[1024] = new char[get_dim()+1][1024];// 其余的维度需要编译时已知常量值
```

### 常量对象的动态分配和释放
当我们希望在自由存储区中创建一个对象，使其具有可控的动态生命期，但是希望这个对象的值不能被改写，则使用如下方式进行对象创建：
```c++
const int *ptr = new const int(1024);// 创建了一个常量int对象
```
创建常量对象需要注意：**1** 常量必须被初始化，因此上面表达式中的括号不能不省略。 **2** new表达式创建常量而返回的指针必须是常量指针。常量必须通过常量指针进行管理，即通过这个指针不能修改其数值；而非常量可以被常量指针管理，此时不能通过这个指针修改数值，但是可以通过其他方式对其修改。 **3** 不能在自由存储区中创建一个内置类型元素为const属性的数组，一个直接的原因是：内置类型数组无法在new表达式创建时被初始化。自定义的类数组可以通过重载new操作符的形式实现在自由存储区中创建const元素属性的数组。
```c++
const int *ptr1 = new const int[1024];// 错误 无法通过new创建常量数组
```

### placement-new：为new表达式在分配内存中构建对象 
通过如下形式的new表达式可以让程序员将对象创建在已经预先分配好的内存中，称之为定位new表达式(placement-new expression):
```c++
#include <iostream>
#include <new>
const int chunk=16;
class foo{
    public:
        foo(){_val=0;}
        int val(){return _val;}
    private:
        int _val;
};
char *buf = new char[sizeof(foo)*chunk];// 预先分配了内存 但是没有创建对象
int main(){
    // new (place_adress) type(specifier);
    foo *ptr = new (buf) foo;// 在预先分配的buf上 创建foo对象
    if(ptr->val()==0){// 验证
        std::cout<<"success!"<<endl;
    }
    delete [] buf;// 不能使用ptr进行释放 内存的管理权在buf手里
}
```
上述程序使用了char数组的形式开辟了一块内存，我们很容易想到：为什么使用char数组开辟这块内存，用于foo对象的构建呢？为什么不选用其他类型？参考stackoverflow中有位大神回答：
> 为什么不使用char呢？char是c++定义中的最小的抽象数据类型，并且在任何抽象机制层面，单个char元素占据1个字节。因此，char类型是一个非常好的用来**衡量内存大小的单位选择**。<br>
> c语言中也提供了具体的机制用来支持这种做法：new char[]的方式申请的内存，将不会按照char类型的方式进行align，而是提供了任何类型maximum normal alignment，这种机制保证了在new char[]开辟的内存空间中能够构造任何类型。

### new & placement-new 
**[new & delete可以被重载]** operator new和operator delete和operator+一致，可以在自定义的类内进行运算符重载。如果类内没有进行new&delete重载，那么默认使用全局的new&delete。`operator new`、`operator new[]`、`operator delete`、`operator delete[]`四个操作符。如果需要自定的版本，推荐的做法是一次性将4个都重载。

**[三种操作符具体做了什么]:** **new**操作符实际完成了两件事：`1` 调用底层malloc()函数完成内存的分配；`2` 调用类型合适的构造函数创建对象。**delete**操作符实际也完成了两件事：`1` 调用析构函数将new创建的对象释放；`2` 调用free释放相应的内存。**placement-new**是对new的一种重载，它只负责调用合适的构造函数在给定的内存位置创建对象并返回控制指针，不负责内存的申请。

**[new]**操作符在使用的过程中需要不断地从自由存储区的链表中查找能够容纳相应对象的内存，这个过程有时会严重影响运行效率，而且会导致内存碎片化。

**[placement-new]**是new的一种重载版本：事先申请存储对象的内存，然通过placement-new表达式将对象放入申请号的内存中，返回用于操作对象的指针。这种内存申请方式不如new灵活，但适用于一下不常更新的稳定运行的功能使用。 

**[内存资源的释放]** new表达式通过delete空间对象的指针来完成，placement-new通过delete申请内存的指针来完成。new内存的释放是对象级别的，而placement-new内存的释放是buffer级别的，尽量保证buffer中对象的生命期相近，可以增加内存的申请利用效率。<br>

### 类类型数组在堆中的动态创建和销毁
**[no.1 类类型数组在堆中的动态创建]** <br>
```c++
typedef vector<pair<char*, double>> vec_pair
Account* Account::init_heap_arr
    (vec_pair &init_values, vec_pair::size_type arr_size){
    vector<value_pair>::size_type vec_size = init_values.size(); 
    if(vec_size==0 && arr_size==0){// 如果不需要分配
        return nullptr;
    }
    // no.1 op-new to apply for memory
    char *ptr = new char[sizeof(Account)*arr_size];
    for(int i=0; i<arr_size; ++i){
        if(i<vec_size){// 根据传入的初始数组构建元素
            pchar_init=init_values.first; double_init=init_values.second;
            // no.2 placement-new as ctor
            new (ptr+offset*i) Account(pchar_init, double_init);
        }
        else{// 剩余的部分执行默认初始化
            new (ptr+offset*i) Account();// no.2 placement-new as ctor
        }
    }
    return (Account*)ptr;// [将ptr类型转化成指向Account对象的指针]
}
```
**[no.2 类类型数组在堆中动态的销毁]** <br>
```c++
void Account::dealloc_heap_arr(Account *p, size_t arr_size){
    for(int i=0; i<arr_size; ++i){
        p[i].Account::~Account();// no.3 call dtor for each ele in arr
    }
    // no.4 release the memory op-new applied
    // [将指向Account对象的指针类型转化成指向char内存的指针]
    delete[] reinterpret_cast<char*> p;
}
```
**[no.3 类类型数组的分配和释放需要互补性操作完成]** <br>
使用**operator-new**申请的内存，应该由**operator-new**返回的指针进行释放，而不是由指向同一块内存首地址的其他指针进行管理。同理，使用**new-expression**申请内存构建对象，应该由**new-expression**返回的指针进行释放，而不是由**operator-new**返回的指针进行释放。两种方式不要混用。
```c++
// new-expression版本的内存对象构建&释放
Account *p=new Account[arr_size];// 使用new-expression: 申请内存 & 构造对象
delete[] p;// 调用析构函数 & 释放相应内存
```
进一步理解**operator-new**和**new-expression**的区别：
```c++
char *p1=new char[sizeof(Account)*arr_size];// [op-new] p指向内存char数组中的位置
Account *p2=new Account[arr_size];// [new-exp] p指向Account数组中的位置
```
两种operator返回的指针不属于相同的类型，虽然他们可能指向同一块内存，因此，不能混用各自的delete语句，并且不能通过**new-expression**返回的指针来完成**operator-new**的收尾工作。

**[将类类型的堆数组分配&销毁函数声明为static]** <br>
最后将上述两个类类型数组动态管理函数声明为Account类的静态函数。将类类型的堆数组声明为Account类的static成员是因为：这两个函数不涉及对Account类调用者的操作，因此不涉及this指针的运用。并且相比普通函数，static函数不需要维护this指针，所以效率会略高。
```c++
class Account{
    public:
        ...
        static Account* init_heap_arr(vec_pair &, vec_pair::size_type);
        static void dealloc_heap_arr(Account *, size_t);
    private:
        ...
};
```




## Reference
> 《c++ primer》3rd p349 <br>
> https://blog.csdn.net/zhangxinrun/article/details/5940019 - placementnew <br>
> https://stackoverflow.com/questions/13370935/c-placement-new - using char to allocate buf <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
