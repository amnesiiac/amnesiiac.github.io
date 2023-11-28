---
layout: post
title: "cpp - type cast"
subtitle: 'c++类型转化机制知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-01 14:20
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---
## implicit conversions
> `promotion转换` Converting to int from some smaller integer type, or to double from float is known as promotion, and is guaranteed to produce the exact same value in the destination type. '算数提升'能够保持相同的数值。<br>

> `truncated转换` If the conversion is from a floating-point type to an integer type, the value is truncated (小数部分被直接移除). If the result lies outside the range of representable values by the type, the conversion causes undefined behavior. <br>
> 从一些'smaller'类型转换成'bigger'类型，称之为'promotion'，不会改变数据值(增加了精度)。

> `负数转换成无符号数` If a negative integer value is converted to an unsigned type, the resulting value corresponds to its 2's complement bitwise representation. <br>
> 负数转换成无符号数，需要用其补码形式进行代替(符号位不变，数值位取反+1)。

> `bool类型隐式转换` The conversions from * to bool consider false equivalent to zero (for numeric types) and to null pointer (for pointer types); true is equivalent to all other values and is converted to the equivalent of 1.

> `没有理解这种转换的意义` If the conversion is between numeric types of the same kind (int->int,float->float,etc), the conversion is valid, but the value is implementation-specific.

> `非基础类型的类型转换:arr & func - 隐式转换成指针` For non-fundamental types, arrays and functions implicitly convert to pointers.

> `NULL & void pointer` NULL pointer可以转换成任何类型；任何类型的指针都可以转换成void pointer。<br>
> 注意NULL pointer和void pointer的不同之处: Void refers to a pointer that points to some data location in storage, which doesn't have any specific type, basically the type of data that it points to is unknown. Null refers to a pointer of any type has such a reserved value, it's essentially a pointer to nothing, and is invalid to use. 即void pointer指的是类型是void，是一个空类型；null pointer指的是值为null，既指向nothing的指针。

> `派生类->基类指针` Pointer upcast: pointers to a derived class can be converted to a pointer of an accessible and unambiguous base class, without modifying its const or volatile qualification. 派生类的指针可以被转换成基类的指针。

> `implicit conversion of classes` 主要由如下三种操作函数进行实现：single-argument constructors，assignment operator，type-cast operator。

```c++
#include <iostream>
using namespace std;

class A{};
class B{
    public:
    // Single-argument constructors:
    // -> implicit conversion from a particular type to init an object.
    B(A const &x){}
    // Assignment operator:
    // -> implicit conversion from a particular type on assignments.
    B& operator=(A const &x){return *this;}
    // Type-cast operator:
    // -> implicit conversion to a particular type. 需调用相应类型的构造函数
    operator A(){return A();}// 从B类对象转换成A类对象的op
};

int main(){
    A foo;
    B bar=foo; // calls constructor
    bar=foo; // calls assignment
    foo=bar; // calls type-cast operator
    return 0;
}
```
> `keyword explicit` `1` It can prevent using the single argument constructor to implement implicit conversion, which may not be our intend. `2` Additionally, constructors marked with explicit cannot be called with the assignment-like syntax; `3` It can prevent implicit conversions in the same way as explicit-specified constructors do for the destination type. <br>

> explicit用来阻止由于定义了single argument constructor, assignment operator以及type cast operator三种函数导致的类型隐式自动转化。具体地：`1`：单参数构造函数中的参数类型可以被隐式的提升成相应的类类型，构造函数被定义成explicit，那么这种隐式的类型转换方式被禁止，显式的直接通过构造函数进行构造的方式不受影响；`2` 构造函数被定义成explicit，那么通过赋值函数进行类型转换被禁止；`3` 自定义类型转化函数为explicit，那么隐式的通过该函数进行类型转化被禁止。<br>

> 总结explicit核心思想，类内定义了explicit关键字之后，首先：看构造函数是否定义了explicit，看有没有自定义的类型转换成员函数；其次：看用户代码中被检查的语句，能够直接调用构造函数，如果能，那么explicit不能使这种**定义了的**转化失效，如果不能，则属于隐式转化范畴，那么语句编译不能通过；最后，下面的例子非常详尽，嫌啰嗦，直接理解例子就好。

```c++
#include <iostream>
using namespace std;

class A{};
class B{
    public:
        // 只有构造函数 和 自定义的显式类型转换函数可以使用explicit
        (explicit) B(const A &x){}
        B & operator=(const A &x){return *this;}// 不能定义为explicit
        (explicit) operator A() {return A();}// 自定义的从B->A的类型转换
};

void fn(B x){}

int main(){
    A foo;
    
    // 调用构造
    B bar(foo);// A->B 调用B类的构造函数 always right

    // 调用 operator=函数
    B bar=foo;// if constructor explicit: 调用operator=进行类型转换被禁止

    // 调用
    bar=foo;// A->B 调用B类operator=构造函数 always right

    // 这种形式 A类没有定义operator=  那么只能隐式的调用了B类的operator A()
    foo=bar;// if constructor explicit: B->A is NOT banned!
    foo=bar;// if operator A() explicit: B->A is banned!

    // 函数传参过程 类型不同则发生隐式转化
    fn(foo);// if constructor explicit: A->B implicit is banned 
    fn(bar);// B->B always right
    return 0;
}
```

## explicit conversions
### static_cast
> 强制类型转化过程中，不需要编译器进行强类型检查，对于转化是否安全完全取决于程序员。也因此，这种转化避免了强类型检查的带来的overhead。<br>
> static_cast可以完成所隐式类型转化，并且比implicit_conversion作用范围更广。

> `void类型conversion` <br>
> void\*类型 ➡ 其他任何指针类型 <br>
> 任何类型 ➡ void类型

> `enum类型conversion` <br>
> integers, floats, enum types ➡ enum types <br>
> enum class values ➡ integers, floats

> `class类型conversion` <br>
> upcasts: pointer-to-derived ➡ pointer-to-base <br>
> downcasts: pointer-to-base ➡ pointer-to-derived

> `confused类型conversion` <br>
> explicitly call a single-arg constructor or a conversion op. <br>
> convert to rvalue references: 转化成右值引用。

### dynamic_cast
**[dynamic\_cast主要用法]** <br>
**(1)** dynamic_cast只能在指针(包括void\*指针)和引用一起使用。<br>
**(2)** dynamic_cast可以进行downcast，也可以进行upcast。<br>
**(3)** dynamic_cast必须和虚函数表(系统生成)结合使用。<br>
**[upcast和downcast基本概念]** <br>
upcasts: pointer-to-derived ➡ pointer-to-base (by static_cast or dynamic_cast)<br>
downcasts: pointer-to-base ➡ pointer-to-derived(polymorphic classes with virtual members): 在含有virtual成员的多态类之间的基类指针转化成派生类的指针。<br>

**[dynamic\_cast使用注意事项]**<br>
"But dynamic_cast can also downcast with polymorphic classes if and only if the pointed object is a valid **complete** object of the target type." <br>
虽然dynamic_cast能够完成downcast操作，但是只有当被cast的指针具有和目标对象类型相同的对象内存空间，才能够正确的cast过去，否则将会返回一个nullptr来指示类型转化failure。另外，如果dynamic_cast被用来转化成一个引用类型，但是转化失败，一个bad_cast类型的exception将会抛出。

```c++
#include <iostream>
#include <exception>
using namespace std;

class Base{virtual void dummy(){};};
class Derived:public Base{int a;};
int main(){
    try{
        Base *p0=new Derived;// 基类的指针 初始化的时候开辟了派生类的内存
        Base *p1=new Base;// 基类的指针 初始化时开辟了基类的内存
        Derived *pd;
        // no.1: 将基类指针(具有complete object空间) -> 派生类的指针
        pd=dynamic_cast<Derived*>(p0);
        if(pd==0){cout<<"Null pointer on first type-cast.\n";}// 不执行
        // no.2: 将基类指针(incomplete object空间) -> 派生类的指针
        pd=dynamic_cast<Derived*>(p1);// pdf=0=nullptr
        if(pd==0){cout<<"Null pointer on second type-cast.\n";}// 执行
    }catch(exception& e){
      cout<<"Exception:"<<e.what();
    }
  return 0;
}
```
**[dynamic_cast和运行时刻类型识别RTTI]** <br>
"Compatibility note: This type of dynamic_cast requires Run-Time Type Information (RTTI) to keep track of dynamic types. Some compilers support this feature as an option which is disabled by default. This needs to be enabled for runtime type checking using dynamic_cast to work properly with these types." <br>
dynamic_cast类型转换需要启动RTTI选项，包含这个功能的编译器在使用的过程中需要手动开启RTTI功能，以保证dynamic_cast正常运行。

**[dynamic_cast的其他用法]** <br>
"dynamic_cast can also perform the other implicit casts allowed on pointers: casting null pointers between pointers types (even between unrelated classes), and casting any pointer of any type to a void\* pointer." <br>

**[dynamic_cast和static_cast的区别]** <br> static_cast和dynamic_cast都可以完成downcast以及upcast。如果用来执行upcast(派生类对象指针->基类对象指针)，两者的区别不大，因为所指向的内存空间具有包容性。<br> 
然而用来执行downcast(基类对象指针->派生类对象指针)，那么dynamic_cast相比static_cast更加安全(使用dynamic_cast的情况下，通常会启用RTTI功能)，dynamic_cast执行运行时类型检查，而static_cast执行编译期类型检查，运行期不做检查。

**[更多关于dynamic_cast和RTTI机制以及类继承之间的关系]**可以参考[博客](/cpp/2021/06/17/post-cpp-inheritance-2/)中的内容进行理解。

### const_cast
> `基本用法` This type of casting manipulates the constness of the object pointed by a pointer, either to be set or to be removed. <br>
> const_cast用来改变`const [type] *pointer=&object`结构中，`object`的属性，即可以为其添加或者删除const属性。

```c++
#include <iostream>
using namespace std;

void print (char *str){cout<<str<<'\n';}// called

int main (){
  const char *c="sample text";
  print(const_cast<char *>(c));// remove constness of pointer c 
  return 0;
}
```

> `const_cast注意事项` The example above is guaranteed to work because function print does not write to the pointed object. Note though, that removing the constness of a pointed object to actually write to it causes undefined behavior. <br>
> const_cast将`const type *ptr=&obj`中的`obj`类型从`const type`类型改成`type`类型，如果程序需要对于`obj`进行写入操作，那么需要谨慎操作，对const对象去掉const属性执行写入操作可能引发未定义的结果。

### reinterpret_cast
> `基本用法` reinterpret_cast converts any pointer type to any other pointer type, even of unrelated classes. The operation result is a simple binary copy of the value from one pointer to the other. All pointer conversions are allowed: neither the content pointed nor the pointer type itself is checked. <br>
> reinterpret_cast用于将任意类型的指针`ptr1`转化成任意类型的指针`ptr2`，其中`ptr2`只是`ptr1`的2进制内存数值上的一份copy。对于指针指向的内容和指针本身不进行任何检查。

> `补充` reinterpret_cast不只适用于指针类型之间的相互转化，如它可以实现int和char*之间的转化，本质上是从底层2进制**重新解释抽象成新的类型**。<br>
> reinterpret_cast是一种底层的强制类型转换，由于这种转换类型的方式，因此它在非底层代码之外应该极为罕见。

补充关于reinterpret_cast的内容，参考[博客](/cpp/2021/05/11/post-cpp-class-template/)中类模版的静态成员中*operator-new*和*operator-delete*的重载。reinterpret_cast一个重要的用法是：将*new char[chunk_size]*分配的原始内存重新抽象成指定类型的待构造内存。
```c++
typedef someclass T
char *p = new char[chunk_num*sizeof(T)];// 申请原始内存
T *ptr = reinterpret_cast<T*>(p);// 将原始内存转化成待构造内存
```

## Reference
> http://www.cplusplus.com/doc/tutorial/typecasting/

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
