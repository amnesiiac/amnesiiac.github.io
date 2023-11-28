---
layout: post
title: "cpp - allocator"
subtitle: 'allocator关键字知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-10 17:30
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---
### 为什么需要使用allocator?
当我们分配单个或者已知数量的对象时，通常希望能够将对象构造和内存分配绑定，因为我们的目标是已知的，这种情况下使用new&delete的静态分配方案有更好的效率。<br>
但是，new & delete存在灵活性的局限：`new expressions`将内存分配和对象构造绑定在一起，`delete`将对象析构和内存释放绑定在一起。
当我们需要分配很多对象，对象的个数是不确定的，对象的类型也是不确定的，那么我们更愿意将内存的申请和对象的构造分开来进行处理。如果使用内存分配和对象构造绑定的方式，那么多余的对象的构造造成了一定的浪费，以及有些对象执行了默认构造，在使用时需要重新构造，初始化时的无用构造造成了资源的浪费。<br>
另外，对于默认构造=delete的类型而言，该类对象的动态分配只能通过内存、对象分开处理的方式完成。

关于`new expressions`和`new operator`的区别，参考[链接1](https://en.cppreference.com/w/cpp/language/new)以及[链接2](https://en.cppreference.com/w/cpp/memory/new/operator_new)。简而言之，new expressions将内存分配和对象构造绑定，在内存分配的时候调用了new operator。allocator同样底层是调用了new operator完成内存分配的。

关于allocator是否需要，网上有很多大神进行了讨论，参考[链接3](https://www.zhihu.com/question/274802525)。

### allocator的基本使用方法
**[1]** 定义一个allocator类的实例，用于存储T类型对象。
```c++
#include <memory>
allocator<T> a;// 定义了一个allocator对象 用于给T类型对象分配内存
```
**[2]** 用allocator对象分配原始未构造的内存。
```c++
T *p = a.allocate(n);// 分配了容纳n个T类型对象的原始内存 返回分配内存首指针
```
**[3]** 用deallocator将allocator申请的内存释放 - allocate逆过程。
```c++
// 完全是allocate的逆过程 n是allocator中的n p是allocator返回的指针
// 调用deallocate之前对这段内存中的对象执行destroy操作
a.deallocate(p, n);// return none
```
**[4]** 在allocate申请的内存中构建对象。
```c++
// p是T类型指针 指向原始内存 args被传递给T类型构造函数 用于构建对象
a.construct(p, args);// return none
```
**[5]** 对于内存上对象执行析构函数。
```c++
a.destroy(p);// 对于T类型指针p指向的对象 调用其析构函数 return none
```

### allocator补充的方法
```c++
// 给定初始化的allocator类对象
allocator<T> a;
auto ptr = a.allocate(n);
```
**[1]** 将迭代器范围内的对象放到ptr指定的内存中，返回尾后指针。
```c++
T *p = unintialized_copy(iter0, iter1, ptr);
```
**[2]** 从iter0开始，拷贝n个对象到ptr指定的内存中，返回尾后指针。
```c++
T *p = uninitialized_copy_n(iter0, n, ptr);
```
**[3]** 在iter0-iter1范围内，全部构造T类型对象t，return none。
```c++
uninitialized_fill(iter0, iter1, t);
```
**[4]** 在iter0处，构造n个T类型对象t，iter0必须能够容纳所构造的对象。
```c++
uninitialized_fill_n(iter0, n, t);// return none before cpp11
T *p = uninitialized_fill_n(iter0, n, t);// 返回尾后指针 after cpp11
```


## Reference
> 《c++ primer》5th 12.2.2 <br>
> https://www.zhihu.com/question/274802525  c++为什么需要allocator类 <br>
> cpp reference 

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
