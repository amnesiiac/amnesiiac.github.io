---
layout: post
title: "cpp - inline functions"
subtitle: 'c++中内联函数知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-04-16 17:40
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals 
---
### 为什么要使用inline 
函数的调用需要：拷贝实参(传值方式)，保存程序寄存器状态，程序跳转到其他位置进行执行。函数调用远比直接计算表达式要慢得多。因此，我们在函数声明的地方，加上inline用于向编译器建议，在编译期将函数`内联的`展开，从而提升程序的效率。
```cpp
IntArray array;
// 通过调用IntArray类中的size()方法来获得array中变量_size信息
int arraysize = array.size();
// 直接通过调用IntArray类中的_size数据成员来获得信息
int arraysize = array._size; 
```
显然，第一个方法需要函数调用，而第二个方法只需要直接访问内存。一般来说，通过调用函数访问内存(栈空间)和直接访问内存相比开销要大很多。在这种情况下，c++提供了一种内联函数的机制，声明为内联的函数在编译的过程中根据建议决定是否展开，并且内联函数不会引起任何函数的调用，用下面的例子能够很好的说明这个问题。
```cpp
// 下面的程序并不会在for循环中每次执行函数体都调用IntArray的size()函数
for(int index = 0; index < array.size(); ++index){...}
// 上面的写法在编译期被以如下的方式'内联的'展开 两种东西等效
for(int index = 0; index < array._size; ++index){...}
```
原因是，在类中定义的成员函数(如上面的size())，在编译期中，会被自动的当成是内联函数进行处理。此外，我们可以使用inline关键字向编译器显式地提供一种内联的建议。

### inline具体做了什么
```cpp
inline int min(){...}
int minval=min(i,j);// 函数调用
int minval=i<j << i:j // 编译器将上述调用进行展开
```
### inline应用规则
`1` inline只能在函数体内部代码较为简单的情况下使用，不能包含复杂的控制语句如while、switch以及超过限制数量的if结构。<br>
`2` 函数存在递归调用的情况下，不能定义成inline函数。<br>
`3` 多次使用的简单小型工具函数一般声明为inline，但是inline会增加代码体积(在所有调用点展开)，所以inline函数应该\<短小精悍\>。<br>
`4` inline只是给编译器的一种建议，编译器忽略了某些inline函数会给出warning提示。<br>
`5` 在程序debug的过程中，inline函数不按照内联方式处理，方便进行debug。<br>
`6` inline函数的声明定义尽量放在头文件中。inline函数会在调用点展开，因此编译器必须\<及时\>看到inline函数的定义，为了避免头文件声明，多个源文件写相同的inline定义(太麻烦)，直接把inline函数定义在头文件中即可。<br>
`7` 定义在类内部的成员函数(缺省inline符号)都是内联的，类内没有给出inline定义，要想使用inline特性，类外定义必须使用inline关键字，否则编译器不按内联处理。<br>
`8` 补充\<7\>：在类内写函数定义虽然省事，但不是好编码风格，应该改成:
```cpp
class test{// 头文件中写类的定义
    public:
        inline void func(int x, int y);// 声明为inline函数
};
inline void test::func(int x, int y){...}// 在<定义文件>中实现函数定义
```
`9 inline参数解析规则` inline函数参数、inline函数定义中的参数在类内的解析顺序：**1** 首先在inline函数在类中出现的位置解析其类型(函数返回类型、函数参数表)。**2** inline函数的定义部分使用的变量在完整的类域定义后面执行。<br>
关于类内声明顺序的拓展：除了inline函数的定义部分，成员函数的缺省实参的解析也是在类域完成后进行(需要注意的是成员函数缺省实参必须是静态类型，如果使用了非静态成员则程序报错)。
类内其他成员需要按照先声明后使用的方式应用。关于static更多用法详见[博客](/cpp/2021/05/06/post-cpp-static/)。
```cpp
class String{
    public:
        typedef int index_type;// 必须出现在op[]函数使用之前
        // 在类内定义的op[]被自动声明为inline
        // 在operator[]声明出解析 char& (index_type) 如果index_type没有声明则报错
        char & operator[] (index_type ele){
            return _string[ele];// 类域中所有的声明对inline函数定义中的变量都可见
        }
    private:
        char *_string;
};
```

### inline用法细节
**Wrong way to declare & use inline** 
```cpp
// 结论 - fool函数调用时没有按照inline方式展开
void fool(int, char*);// 函数没有声明成inline
fool(i, arr);// 函数调用在声明之后 定义之前
inline void fool(int i, char *arr){...}// 函数定义在调用之后 被定义成inline
```
**Right way to declare & use inline** 
```cpp
// 结论 - 只有声明、定义都在调用之前 使用inline关键字 才能内联
inline void fool(int, char*);// 声明fool为inline函数
void fool(int i, char *arr){...}// 定义时没有定义成inline
fool(i, arr);// 在调用时不会按照inline方式打开
```

## Reference
> 《c++ primer》

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
