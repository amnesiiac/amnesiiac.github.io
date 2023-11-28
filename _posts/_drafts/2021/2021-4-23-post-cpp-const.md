---
layout: post
title: "cpp - const"
subtitle: 'const关键字知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-04-23 10:27
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---

### 类对象的const属性
一般来说：常量类对象调用常量(this)版本的重载成员函数，非常量对象调用非常量版本的重载成员函数。但是，这个规则在构造函数和析构函数上的应用有例外：<br>
类内的构造函数和析构函数的调用中，对象的常量性不能决定调用'常量'构造函数(构造函数不能被声明const this)：常量对象和非常量对象都可以调用同一构造函数，准确的说，当对象构造完成之后，它的const属性才被建立起来。常量对象和非常量对象都可以调用同一析构函数，准确的说，当对象调用析构的时刻，它的const属性已经被拿掉。简而言之：对象的const属性只在从对象构造完成到对象析构之前存在。

### c_str()和const的关系
```c++
#include<string>
using namespace std;
string s1;
char *c1;
s1 = c1; // 这个赋值是正确的 string中包含了从char向string类型的转化
c1 = s1; // 这个赋值是错误的 char不能接受这种类型转化
c1 = s1.c_str(); // 错误 c_str()只能将string转化成const char*
const char *c2; c2 = s1.c_str(); // 正确 两侧类型相互匹配
```

### const的read-only属性 
const类型限定修饰符的一个作用是：将被修饰的对象转化成一个常量(constant)，即对象是read-only的。
```c++
const buffer_size = 512; // 任何企图修改buffer_size的操作都是编译错误的
```
另外，常量在定义之后，只读且不可修改，因此，常量必须在定义的时候进行初始化，未初始化的常量导致编译错误。
```c++
const int buffer_size2; // 错误 常量定义的时候没有初始化
const string buffer_str; // 正确 string类型含有default构造 能够执行默认初始化
```
试图将一个非const对象的指针指向一个常量对象的操作会引起编译错误。
```c++
const double *cptr = 0;
const double wage = 9.60;
cptr = &wage; // 正确的 我们可以将一个指向常量的指针 指向一个常量
double *ptr = 0;
ptr = &wage; // 错误的 不能将非常量指针 指向一个常量
```

虽然非常量的指针ptr不能够指向一个常量，但是常量指针cptr可以指向一个非常量，参考下面的例子。
```c++
const double *cptr = 0;
const double wage = 9.60;
double wage2 = 9.60;
cptr = &wage; // 正确 常量指针可以指向一个常量 不能通过cptr修改wage数值
cptr = &wage2; // 同样正确 常量指针可以指向一个非常量 不能通过cptr修改wage2数值
wage = wage+1; // 错误 wage本身是常量 不能修改 
wage2 = wage2+1; // 正确 wage2本身是非常量 可以修改
```
上面这个例子的一个经典用法是，通过const指针向函数传递参数。
```c++
// 使用const类型指针方式传递参数 可以保证被传递给函数的实际对象不被修改
int strcmp(const char *str1, const char *str2);
```

### 常量指针 指针常量 常量指针常量
**[定义]** const是一个qualifier修饰词，在理解const修饰关系的时候，可以认为const有就近修饰原则。
```c++
// 常量指针
const double *cptr; 
// 首先 const修饰的是double 表明数据类型是double 并且数据是一个constant
// 然后 *cptr表明这是一个指针定义 表示cptr指向前面定义的数据
// 需要注意的是 cptr本身并不是一个costant量 因此cptr中的地址值可以修改

// 指针常量
double *const cptr;
// 按照const就近修饰原则 const修饰的是cptr 表明cptr是一个常量(指针常量)
// double *表明 cptr是一个指向double数据类型的指针常量

// 常量指针常量
const double *const cptr;
// 按照const就近修饰原则 第一个const修饰的是double 表明cptr指向一个常量
// 第二个const修饰的是cptr 表明cptr是一个指针常量
```

**[常量指针使用方法 - 及注意事项]** 常量指针可以用常量的地址进行定义，也可以使用非常量的地址进行定义，而非常量可以通过其他方式进行值的修改，但是，不能通过一个指向它的常量指针进行修改。详见下面的代码：
```c++
int num = 1;// 定义一个非常量
int *ptr = &b;// 非常量指针定义
const int *const_ptr = &b;// 常量指针定义
*ptr = 2;// 通过非常量指针改变num的值 -> 正确!
*const_ptr = 2// 通过常量指针改变num的值 -> 错误!
```
另外，在声明定义类的成员函数时，如果不想通过该成员函数修改类对象，应该将成员函数的this指针定义为常量指针。一种例外的情况是：当类的数据成员中含有指针类型，那么const虽然限制了不能通过this指针修改类的对象，但不能阻止通过直接修改this指向的内存的方式进行数据修改：
```c++
class Text{
public:
    void func(const string &) const;// const修饰this func不能修改Text数据成员
private:
    char *_text;// 私有指针成员
};
void Text::func(const string &str) const{
    _text=str.c_str();// 错误 const this 不能直接修改私有数据成员_text
    for(int i=0; i!=str.size(); ++i){
        _text[i] = str[i];// 正确 能直接修改_text指向内存中的数据 不推荐的编码风格!
    }
}
```

**[常量指针在全局作用域下的声明定义]** 在默认情况下，c++中的全局变量的链接性为外部的，而const限定的命名空间中的变量处理为内部链接性(internal linkage)，又称为静态链接性(static linkage)。具有内部链接性的变量在编译后，不会产生链接符号，因此不和链接器打交道，因为它们具有内部的(自发的)链接性。<br>

通常常量指针的声明和定义有两种方式：**方式1** 在头文件中声明，在所有使用这个变量的文件中进行重复相同的定义，以保证在使用时编译器能够看见它的定义。**方式2** 在头文件中声明，并且在头文件中定义，这种方式更加清晰，明确，推荐使用这种写法。

因此，被const修饰的变量可以定义在头文件中，供工程中其他文件使用，编译时不会产生多重定义的错误。特别的，当const修饰指针的时候，需要注意：
```c++
// 头文件CONST_STRING_H_中的定义 头文件被多个工程文件包含 ->
onst char * const_str = "jackie chen";// 常量的指针 外部链接性 错误!
char * const const_str = "jackie chen";// 指针常量 内部链接性 正确!
const char * const const_str = "jackie chen";// 常量的指针常量 内部链接性 正确!
const char str_arr[] = "jackie chen";// 数组名是常量的指针 外部链接性 错误!
```
注意，内部链接性的变量才能够在头文件定义，多个工程文件包含(工程文件不属于同一编译单元即可)，不产生链接错误(多重定义的错误)。此外，被inline修饰的函数也具备内部链接的性质，和常量指针一致的，inline修饰的函数也有两种方式进行声明和定义。inline对编译器是一种建议，当编译器忽略inline请求时，会在每个使用这个函数的文件中为其补充一份函数定义。关于inline的更多的特性，可以参考[这篇博客](/cpp/2021/04/16/post-cpp-inline/)。关于内部链接性和外部链接性的内容，可以参考[这篇博客](/cpp/2021/05/06/post-cpp-linkage/)。


## Reference
> 《c++ primer》- N.Gregory Mankiw <br>
> wikipedia - 链接性 

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
