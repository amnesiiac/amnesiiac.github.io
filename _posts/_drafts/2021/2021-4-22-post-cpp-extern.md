---
layout: post
title: "cpp - extern"
subtitle: 'c++中extern关键字知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-04-22 20:40
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---
### 用法一:标示变量在外部定义
变量的定义会引起相关内存的分配，每一个对象只能有一个位置，程序中每个对象只能被定义一次。如果我们需要在某个变量定义文件之外的地方使用该变量，则可能会引发下面的问题。

```c++
// file module0文件下 定义变量filename
string filename;
// file module1文件下 想要使用filename
filename_new = filename + "appendix"
// 此时 编译器会报错 在file module1文件中 filename未定义 
```

那么为了能够让module1文件编译通过，需要将module0文件中的filename定义引入进来，但是又不能够重复定义filename，因此，采用下面的代码可以解决这个需求，它说明了在程序的外部含有filename的定义。

```c++
// file module1文件下 使用extern关键字进行对于filename变量给予声明
extern string filename // 这行代码不是filename的定义 是一个声明 不会引起内存分配
```

### 用法二:链接指示符
extern关键字除了上述的对于外部的变量在当前文件的使用加声明之外，还可以放在函数名之前，用来告诉编译器，这个函数名要按照c风格去翻译，而不是按照默认c++的方式去翻译。
```c++
// 普通声明的函数 在编译器完成编译的时候 会在原有函数名func上按照一定的规则'添油加醋'
// 这种机制应用在 重载的多个同名函数在编译器视角下保持唯一的命名
void func(int a, int b);
```
```c++
// 下面这种含有extern的写法 相当于指定编译器按照c的风格去进行编译函数func
extern "C" void func(int a, int b);
```

**[单一语句形式的链接指示符]**
```c++
extern "C" void exit(int);// 假定exit是c语言写的
```

**[复合语句形式的链接指示符]**
```c++
extern "C" {int printf( const char* ... ); int scanf( const char* ... );}
```

**[复合语句包含include]**
```c++
extern "C" {#include <cmath>}// include头文件中所有的声明都被认为是c语言写的
```

**[链接指示符不能出现在函数体中]**
```c++
// 链接指示符不能放在函数体中 - 编译错误
void main(){
    extern "C" void exit(int);
}
// 正确的写法 - 声明于全局域
extern "C" void exit(int);
void main(){exit();}
```

**[链接指示符出现在函数第一次声明中，则后续所有声明都接受第一个声明中的规则]**
```c++
// headfile.h
extern "C" double calc(double);// 第一次声明1
// sourcefile.c 
#include "headfile.h"// 再次声明2
double calc(double dparm){...} // calc()可以在C程序中被调用
```
**[补充说明：关于链接指示符的用法]** 链接指示符相关的声明放在工程文件中的头文件更加合适。

**[补充说明：关于extern "C"名字修饰相关内容]** c++语言在编译的时候为了解决函数的多态问题，会将函数名和参数联合起来生成一个中间的函数名称，而c语言则不会，因此会造成链接函数名和写程序时不一致的情况，此时c语言的函数就需要用extern "C"进行链接指定，这告诉编译器，请保持我的名称，不要给我生成用于链接的中间函数名。这部分内容称之为名字修饰，[wikipedia](https://zh.wikipedia.org/wiki/名字修饰)对此有简单的介绍。

**[补充说明：关于extern "X"形式声明其他语言函数]** `extern "Ada"`可以用来声明是用`Ada`语言写的函数；`extern "FORTRAN"`用来声明是用 `FORTRAN`语言写的函数。


### 用法三：extern "C"和重载函数
**[1]** 链接指示符在一组重载函数中的声明方式：重载函数中只有一个函数可以被声明成extern "C"类型，这是因为extern "C"链接指示符不使用类型安全链接(type-safe-linkage)机制，在编译器的底层，不会通过名字修饰将重载的函数按照不同的类型分别命名。如果一组重载函数中有多个函数声明了extern "C"，那么将会出现编译器底层名字冲突。
```c++
// 错误! 在一组重载函数中 只能有一个函数被声明为 extern "C"
extern "C" double compute(double);
extern "C" int compute(int);
// 正确
extern "C" double compute(double);// 这个函数既可以被c语言调用 也可以被c++调用
int compute(int);// 这个函数只能被c++调用
```
**[2]** 链接指示符不能决定调用重载函数那个版本，调用被重载的函数时，参数表的类型决定了如何调用。
```c++
compute(27.2);// 调用c语言写的compute(double)
compute(10);// 调用c++语言写的compute(int)
```

## Reference
> 《c++ primer》- N.Gregory Mankiw <br>
> https://zh.wikipedia.org/wiki/名字修饰 <br>
> https://www.cnblogs.com/yc_sunniwell/archive/2010/07/14/1777431.html

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
