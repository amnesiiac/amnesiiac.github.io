---
layout: post
title: "cpp - volatile"
subtitle: 'c++中volatile关键字知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-04-24 20:16
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---
### volatile限定修饰符简介
`c++ primer 3nd p104`
当一个对象的值可能在编译器的控制或者检测之外被改变时，如一个能够被系统时钟所改变的变量，则该变量应该声明成volatile类型。因此，编译器执行的某些例行优化不能在volatile类型对象上应用。
```c++
// volatile限定符对于对象的修饰方式和const类似
volatile int display_register;// display_register是一个int型volatile对象
volatile task *curr_task;// curr_task是一个指向task型volatile对象的指针
volatile int ixa[max_size];// ixa是一个数组 数组元素都是int型volatile对象
volatile screen bitmap_buf;// bitmap_buf是一个自定义screen类型的对象 
// 对象的数据成员都是volatile的
```
volatile的作用是，提醒编译器，该对象的值可能在编译器未检测到的时间被改变，因此编译器不能武断地对该对象进行优化，并且在读写过程中都会从变量地址直接读取数据。如果没有声明volatile，编译器可能会优化读取和存储，可能会暂时使用寄存器中的数值。

### volatile应用实例
总结volatile使用方法。(1) 中断服务程序中修改的其他程序中的变量，应当使用volatile。(2) 多任务共享环境下，任务共享的标志需要声明为volatile。(3) 存储器映射的硬件寄存器需要加volatile说明，因为每次对于寄存器的读写可能有不同的意义。
```c++
static int i=0; // 定义1
volatile int i=0;// 定义2
int main(void){
    while(1){
        // 编译器会判断i的数值 决定是否执行do_something函数
        // 但是 定义1：编译器判断在main函数中没有修改过变量i 每次只从寄存器中读取i的数值
        // 结果是i=0 do_something函数永远不会被执行
        // 如果按照定义2：编译器会限制对于变量i的读写优化
        if(i){
            do_something();
        }
    }
}
void interrupt(void){// 中断程序 用来改变变量i的数值
    i=1;
}
```

## Reference
> 《c++ primer》- N.Gregory Mankiw <br>
> https://blog.csdn.net/weixin_44363885/article/details/92838607

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
