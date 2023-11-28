---
layout: post
title: "cpp - bit field"
subtitle: 'c++位域相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-28 17:38
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## Bit Field
位域是一种特殊的类的数据成员。位域被声明用来存放特定数量的二进制位(bit)。位域中的数据类型必须是**非静态的integral类型or枚举类型**。当一个程序需要像其他程序或设备传递二进制位信息时，通常会用到位域。 <br>
关于有序类型：在[static知识整理博客](/cpp/2021/05/06/post-cpp-static/)：类的static const integral(有序类型)成员函数可以在类内进行初始化，但是在类外仍然需要补充定义(不含初值)。

### 位域的定义方式
```c++
typedef unsigned int BIT;
class File{
    public:
        // 在类体相邻位置定义的位域:
        // 编译器尽量将他们放在同一个integral存储位置的相邻位 以进行存储压缩
        BIT mode: 2;
        BIT modified: 1;
        BIT prot_owner: 3;
        BIT prot_group: 3;
        BIT prot_world: 3;
        // 位域在类体的声明方式 常量表达式用于指定数据成员占用的2进制位数
        <Integral-Type> <Name>: Const-Expression
};
```

### 位域的访问&使用方式
```c++
// no.1 定义如下类成员函数用于操作位域成员 或根据位域进行其他操作
void File::write(){
    // 位域的访问限定模式和普通的数据成员一致
    modified = 1;
}
void File::close(){
    if(modified){
        ...// do sth
    }
}
// no.2 定义如下类成员函数 用于判断特定位域标识位的状态 从而允许将位域成员定义为private
inline int File::isRead(){ 
    return mode & READ;
}
inline int File::iswrite(){
    return mode & WRITE;
}
```

### 使用位域的一些注意事项
**[1]** 位域只能是integral有序类型或者枚举类型，位域不能是类的static数据成员。<br>
**[2]** 取地址操作符不能应用到位域上:因为位域有自动压缩内存的机制，多个位域数据成员可能压缩存入同一个可寻址的地址范围内。<br>
**[3]** 由于位域不能使用取地址操作符，因此没有指向位域对象的指针。<br>
**[4]** c++标准库提供了一个处理位的模版类:**bitset**，在可能的情况下，尽量使用bitset而不是位域。

## Reference
> c++ primer 3rd chapter 13.8 p544 <br>
> c++ primer 5th chapter 19.8.1 p756



> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
