---
layout: post
title: "cpp - copy elision"
subtitle: 'c++ copy elision机制知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-04-29 13:40
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---

### copy elision定义
copy elision拷贝省略，顾名思义用于舍弃copy构造以及move构造函数。这小节的内容比较抽象，需要实际操作过程中进行结合理解。后续的理解随时向这里补充，本文是粗略看了一遍reference的结果。

### 强制使用copy elision的情况
> Under the following circumstances, the compilers are required to omit the copy and move construction of class objects, even if the copy/move constructor and the destructor have observable side-effects. The objects are constructed directly into the storage where they would otherwise be copied/moved to. The copy/move constructors need not be present or accessible. <br>
> 在下面的情况下，编译器强制省略copy/move构造函数，即使这么做可能会导致副作用。copy/move构造函数甚至可以不提供或者不能被访问到。

```c++
// 第一种情况 - 返回的操作数是一个相同类型的纯右值 -> 强制执行copy elision
T f(){
    return T();// operand is a prvalue of the same class type 
}
f();// 只调用default构造 不调用copy/move构造
```
```c++
// 第二种情况 - 初始化一个对象的时候 初始化表达式是一个同类型prvalue纯右值
T x = T(T(f())); // 只调用一次default构造 用于初始化T类对象x
// T x = T( T( f() ) ); 这其实就是一个init-expression的嵌套 
// 注意 f()返回非引用类型 根据[value-categories] 是一个prvalue
```
```c++
struct C{ /* ... */ };// definition of C
C f();
struct D;// declaration of D
D g();// declaration of g()
struct D:C{
    // no.1: no elision when initializing a base-class subobject
    // 这里提到的subobject其实就是member object(成员对象) 
    // 成员对象: 一个类的成员是另一个类的对象
    // 构造函数初始化表中初始化一个基类的成员对象 不能使用elision机制
    D():C(f()) {}
    // no.2: no elision because the D object being initialized might
    // be a base-class subobject of some other class
    // 使用下面构造函数初始化的D类对象可能是其他基类(除了C类)的成员对象 即构造
    // 的对象可能被重用 不使用elision机制
    D(int):D(g()) {} 
};
```
总结，使用copy elision要保证：省去了多余的临时变量构造过程，这个临时变量不会对于其他位置的程序结果产生影响。编译器确认了安全之后，才执行elision机制。上面第二种情况中，no1/no2都对于外部产生影响，因此，不执行elision机制。

### 非强制使用copy elision的情况
> Under the following circumstances, the compilers are permitted, but not required to omit the copy and move (since C++11) construction of class objects even if the copy/move (since C++11) constructor and the destructor have observable side-effects. The objects are constructed directly into the storage where they would otherwise be copied/moved to. Even when it takes place and the copy/move (since C++11) constructor is not called, it still must be present and accessible (as if no optimization happened at all), otherwise the program is ill-formed. <br>
> 在下面的情况下，编译器不会强制省略copy/move构造函数操作，这种情况下，无论是否执行了elision，copy/move构造函数必须是present & accessible。

`1` In a return statement, when the operand is the name of a non-volatile object with automatic storage duration, which isn't a function parameter or a catch clause parameter, and which is of the same class type (ignoring cv-qualification) as the function return type. This variant of copy elision is known as NRVO, "named return value optimization". <br>
函数返回表达式，表达式不是一个volatile对象，并且表达式具有自动存储的生命期，且表达式不是一个函数的参数，或者catch语句的参数时，并且返回表达式和函数返回类型一致，这种情况称之为named RVO(具名的返回值优化)。

`2` In the initialization of an object, when the source object is a nameless temporary and is of the same class type (ignoring cv-qualification) as the target object. When the nameless temporary is the operand of a return statement, this variant of copy elision is known as RVO, "return value optimization". <br>
在初始化一个对象过程中，当源对象是一个无名同类型的临时变量，当无名临时变量作为函数返回操作数时，调用RVO机制进行编译器优化。

`3` In a throw-expression, when the operand is the name of a non-volatile object with automatic storage duration, which isn't a function parameter or a catch clause parameter, and whose scope does not extend past the innermost try-block (if there is a try-block). <br>
在throw表达式中，当操作数时一个具有自动存储生命期的非volatile对象时，并且不是函数参数或者catch语句的参数，作用域不超过try作用块的时候，调用RVO。

`4` In a catch clause, when the argument is of the same type (ignoring cv-qualification) as the exception object thrown, the copy of the exception object is omitted and the body of the catch clause accesses the exception object directly, as if caught by reference. <br>
在catch语句中，当参数和exception对象类相同时，exception对象拷贝给参数的过程被omited，直接以引用的方式传值。

`5` In coroutines, copy/move of the parameter into coroutine state may be elided where this does not change the behavior of the program other than by omitting the calls to the parameter's constructor and destructor. This can take place if the parameter is never referenced after a suspension point or when the entire coroutine state was never heap-allocated in the first place. <br>
这个没看懂......主要是不懂得coroutines(协程)的理论知识。


## Reference
> https://en.cppreference.com/w/cpp/language/copy_elision

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
