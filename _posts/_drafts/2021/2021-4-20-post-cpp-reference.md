---
layout: post
title: "cpp - reference"
subtitle: 'c++中引用相关知识总结'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-04-20 23:50
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---
### 引用=别名
```c++
IntArray& rhs;
rhs._size; // 这种写法是正确的 引用的操作方式和引用的实体相同
rhs->size(); // 这种写法是错误的 不能以指针的方式操作引用
```
引用相当于对象的别名，是对象的另一个名字，所以对于引用名字的操作就相当于直接操作对象，对引用的修改相当于直接修改对象。使用方式和指针类似，但是语法上不同于指针。和指针一样，引用提供了对于对象的间接访问。


### 引用必须被初始化
```c++
int ival=1024;
int &refval=ival; // 引用由类型标示和取地址符构成 正确 引用必须被初始化
int &refval2; // 错误 引用必须被初始化

int *pi=&ival;
int *&refval=pi; // 正确 使用引用 注意等式两边类型一致 pi(int *) &revval(int *)
```


### 常量引用 - 常量的引用
```c++
double dual=3.14;
// 直接引用(跨类型)
int &ref_dual=dual; // 错误 引用类型不一致
// const引用(跨类型)
// const int & 和 int const &没有区别 -- csdn 陈硕
// 我喜欢使用 int const &的方式去理解 可以套用const就近修饰
const int &ref_dual=dual; // 正确 只要能通过类型转化 可以使用const引用
int const &ref_dual=dual; // 正确 这种方式和上面的方式相同 属于const引用
// const引用(引用文字常量)
double &ref_dual=dual+1.0; // 错误 dual+1.0是rvalue 不能直接引用
double const &ref_dual=dual+1.0; // 正确 采用const引用可以引用rvalue文字常量
```
进一步地，详细地对于上述const引用使用情况进行解释。
```c++
double dual=3.14;
int const &ref_dual=dual;// 引用存在类型转换
// 此时 编译器将首先生成一个临时对象
int temp=dual; // 此时已经发生了类型转换
int const &ref_dual=temp; // ref_dual实际上取的是临时变量的别名
// 因此 修改ref_dual并不会改变原变量dual 而是会改变temp 
// 通过临时变量的机制 const引用表明了不会改变引用的对象实体 作用如其名 
// 这也是为什么c++规定了 普通的引用方式无法处理‘类型转化引用’以及‘文字常量引用’的原因
```
下面，通过一个更加复杂的例子来理解const引用的使用。
```c++
const int ival=1024;
// 目标：我们想要创建一个&ival的引用 其中&ival表示取ival对象的地址
// 第一种写法
int * &ref_ival=&ival; // 错误 右边是const int *类型 左边是int *类型的引用
// 第二种写法
const int * &ref_ival=&ival; // 错误 右边&ival是一个地址常量 需要const引用才行
// 注意 第二种写法的const修饰的不是& 而是int*指针所指向的对象 因此不是一个const引用
// 第三种写法
const int * const &ref_ival=&ival;// 正确 使用了const引用 并且保证了两边类型一致
// 注意 const &ref_ival表示 &ref_ival是一个const引用 而且引用的是一个const int *对象
```
**<用下面的程序对于引用、常量引用用法进行总结>** <br>
**一：引用的常用写法:** &nbsp; [type] + [引用符号&] + [引用名称ref] = [被引用对象]; <br>
**二：构建步骤** <br>
**[1]** 看被引用对象的类型 const int *, 给予[type]得到: <br>
&emsp; &emsp; &emsp; &emsp; &emsp;[const in *] + [引用符号&] + [引用名称ref] = [被引用对象] <br>
**[2]** 观察是否需要常量引用? &ival是一个地址常量 -> 需要 <br>
&emsp; &emsp; &emsp; &emsp; &emsp;[const int *] + [const] + [引用符号&] + [引用名称ref] = [被引用对象] <br>
**[3]** 问题:const引用const位置如何确定? <br>
**三：关于常量引用的写法:** <br>
**[写法1]** &nbsp; &nbsp; &nbsp;[const] + [type] + [&ref] = [被引用对象] <br>
**[写法2]** &nbsp; &nbsp; &nbsp;[type] + [const] + [&ref] = [被引用对象] <br>
两种写法是等价的，推荐统一使用第一种写法，关于const修饰引用或者指针的问题，可以简单的总结为：const关键字具有**就近**修饰的原则，而且const修饰不能跨越**引用**以及**指针**。
```c++
const int a=1;
const int &ref=a;// const修饰int
int & const ref=xxx;// const修饰ref -> 引用常量 是个错误的概念 
const int *ptr=&a;// const修饰int
int * const ptr=&a;// const修饰ptr
const int * &ref_ival=&ival;// const修饰int 不能逾越指针
const int * const &ref_ival=&ival;// const修饰int 第二个const修饰&
```

### 引用和指针的区别
**[区别一]** <br>
引用总是会指向一个对象，引用在定义的时候必须初始化。
为什么引用必须要初始化呢，这是因为引用的对象必须是一个`lvalue(左值)`或者`xvalue(消亡值-即右值引用)`，即使是const引用，也需要构建一个临时变量来充当左值供const引用使用。如果不提供给引用一个左值，那么引用类型的默认构造函数无法自动创建一个左值出来。
> 关于lvalue rvalue等，参考：https://en.cppreference.com/w/cpp/language/value_category

**[区别二]** <br>
引用是一种特殊的指针，引用之间的赋值和指针之间的赋值在操作中略有不同，结果上都是换绑定。
```c++
int ival=1024; int ival1=2048;
int &ref_ival=ival; int &ref_ival1=ival1;
ref_ival=ref_ival1;// 引用间的赋值 不会改变引用原绑定对象的内容 但改变引用的绑定情况
int *p_ival=&ival; int *p_ival1=ival1;
p_ival=p_ival1;// 指针间的赋值 改变指针本身p_ival重定位到p_ival1指针指向的对象
```
**[区别三]** <br>
c风格数组作为参数传递给函数时，传递引用相比传递指针，可以省略数组长度。见如下代码:

```c++
// 下面的三个声明是等价的
void putValues(int[10]); -> void putValues(int *); -> void putValues(int[]);
// 上面的定义有一个问题：数组的长度 函数不知道 编译器也不知道 -> 解决:
void putValues(int *, int size);// 方法一 指针+长度 
void putValues(int (&ref)[size]);// 方法二 数组的引用 - 省略了数组长度
```
通过传递一个指定长度的数组类型的引用，可以省略长度参数。即当函数参数是数组类型的一个引用时，数组长度是参数类型的一部分。<br>
注意，引用的数组和数组的引用写法上的区别：

```c++
int & ref[10] <=> (int &) ref[10];// 表示引用的数组 - ref是一个数组名
int (&ref)[10]; // 表示数组的引用 - ref是一个数组的引用名
```

### 数组的引用和引用的数组
**[引用的数组]** 基本元素是引用的数组。**[1]** 引用的数组不能作为函数参数传递。**[2]** 引用需要被初始化。**[3]** 引用的数组无法简单的进行初始化。**[4]** 引用是别名，编译器无法确定分配多少空间给引用的数组。**[5]** 不要轻易使用引用的数组，请用指针数组。

```c++
int & arr[10]; // 1
(int &)arr[10];// 2
```
**[数组的引用]** 数组的引用可以使得，在向函数传递数组的过程中，少传递一个数组长度参数，见下面的程序。

```c++
#include <iostream>
using namespace std;

void func(int (&pref)[3]){pref[0]=-1;}// 定义数组的引用
void main(){
	int arr[3]={0, 1, 2};
	func(arr);
	for(int j=0; j<3; ++j){
        cout<<arr[j]<<"  ";
	}
}
```


### 何时使用引用何时使用指针?
两种机制都允许函数修改实参指向的对象。不同的是：<br>
**[必须使用指针]**&emsp;指针可以即时取用，而引用时时刻刻都需要指向一个对象，当参数不希望它指向任何对象时，只能使用指针。<br>
**[建议使用引用]**&emsp;引用使得实现操作符重载的同时，保持用法的直观性，如下例：<br>
```c++
// [value version]
matrix::operator+(matrix m1, matrix m2);// 出现多余copy 运行低效!
// [pointer version]
matrix::operator+(matrix *ptr1, matrix *ptr2);// 使用指针 高效不直观!
matrix a,b,r; r=&a+&b;// 这种写法非常难看!
// [ref version]
matrix::operator+(matrix &ref1, matrix &ref2);// 使用引用 高效直观!
matrix a,b,r; r=a+b;// 重载的写法和原生写法一致!
```

### 函数返回值和返回引用的区别探讨
c++函数返回大致可以划分成两个阶段：返回阶段和绑定阶段。返回和绑定两个过程都有可能调用拷贝构造函数(copy constructor)，或者调用移动构造函数(move constructor)，来创建临时对象。

实际应用中，需要具体情况具体分析。下面给出一个类别的定义，方便后续对于构造函数调用情况进行探讨。 
```c++
class myclass{
    public:
        myclass(){
            std::cout<<"construct"<<std::endl;
        }
        myclass(myclass const &){// 传入常量引用 因为构造过程不涉及修改传入参数
            std::cout<<"copy"<<std::endl;
        }
        myclass(myclass &&){// 传入一个非常量右值引用 因为可能需要销毁共用指针
            std::cout<<"move"<<std::endl;
        }
        ~myclass(){
            std::cout<<"destruct"<<std::endl;
        }
};
```
函数调用阶段分成10种情况分别讨论。
```c++
// 第一种情况 - 生命期较短的变量构建其他对象 优先使用move构造
myclass func1(){
    // 调用default构造创建局部变量mc
    myclass mc;
    // 调用move构造创建临时变量 将mc资源move到临时变量中 调用析构函数析构mc
    return mc;
}
int main(){
    // 调用move构造创建result1 将临时变量资源move到result1中 调用析构函数析构临时变量
    myclass result1=func1();
    // 析构result1
    return 0;
}
// 理论输出: construct \n move \n destruct \n move \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 with '-fno-elide-constructors' 
// 实际输出: construct \n destruct \n
// 解释: x86-64 gcc 10.3 编译器会自动做优化 将函数返回值和main函数中调用move构造进行
//      对象初始化的过程中 临时变量的构建和销毁省略
// 补充: 导致实际输出和理论输出不一致的原因是:c++编译器对于函数返回值的情况使用了
//      return value optimization(RVO)返回值优化以及named RVO(NRVO)具名返回值优化。
//      另外，copy elision可以解决部分copy构造省略的情况 / 实际上 func1的例子可以调用
//      move构造也可以调用copy完成 都定义的情况下 优先使用move
```
```c++
// 第二种情况 - myclass &转化成myclass类型 使用copy构造
myclass func2(){
    // 调用default构造创建局部对象mc
    myclass mc; 
    // 不调用任何构造函数
    myclass &ref=mc; 
    // 调用copy构造根据ref重新构建临时变量(分配新的资源) 调用析构函数析构mc
    // 为什么调用copy构造而不是move构造? 因为ref和返回类型不一致 需要const&转化!
    return ref;
}
int main(){
    // 调用move构造使用临时变量的资源创建result2 调用析构函数析构临时变量
    myclass result2=func2();
    // 调用析构函数析构result2
    return 0;
}
// 理论输出: construct \n copy \n destruct \n move \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 with '-fno-elide-constructors' 
// 实际输出: construct \n copy \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 编译器会自动做优化 func2函数返回情况无法应用NRVO优化 
//      但是在main函数调用临时变量使用move构造result的过程可以优化
// 补充: 函数返回局部变量的引用 这种做法是不推荐的 函数结束后 局部变量mc被销毁
//      但是这样做不会产生副作用 在返回过程中 生成了一个可析构的临时变量进行使用
```
```c++
// 第三种情况 - 同第二种情况 myclass &->myclass 使用copy构造
myclass func3(myclass &para){
    // 调用copy构造使用para资源创建临时对象 为什么调用copy而不是move? ->func2
    return para;
}
int main(){
    // 调用default构造创建'伪全局'global_mc3
    myclass global_mc3; 
    // 使用临时变量调用move构造创建result3 调用析构函数析构临时对象
    myclass result3=func3(global_mc3);
    return 0;// 调用析构函数析构result3 临时对象  global_mc3 顺序和入栈相反
}
// 理论输出: construct \n copy \n move \n destruct \n destruct \n desturct \n
// 解释: x86-64 gcc 10.3 with '-fno-elide-constructors' 
// 实际输出: construct \n copy \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 编译器会自动做优化 func2函数返回情况无法应用NRVO优化 
//      但是在main函数调用临时变量使用move构造result的过程可以优化
// 补充: 函数返回全局变量的引用 实际返回的是全局变量的copy 事与愿违 浪费了全局二字
```
```c++
// 第四种情况 - 不涉及类型转化的引用返回 不调用任何构造函数
myclass & func4(){
    // 调用default构造创建局部变量mc
    myclass mc; 
    // no.1 相当于 myclass & xxx=mc 不调用任何构造函数 调用析构函数析构mc
    return mc;
    // no.2 相当于第一种情况的显式方法 不调用任何构造函数 调用析构函数析构mc
    myclass &ref=mc; 
    return ref;
}
int main(){
    // 调用copy构造通过ref创建result4 为什么调用copy而不是move? goto->func2
    myclass result4=func4();
    return 0;// 调用析构函数析构result4
}
// 理论输出: construct \n destruct \n copy \n destruct \n
// 解释: x86-64 gcc 10.3 with '-fno-elide-constructors' 
// 实际输出: construct \n destruct \n copy \n destruct \n
// 解释: x86-64 gcc 10.3 编译器会自动做优化 但是此情况没有copy elision优化的空间 
// 补充: 返回局部变量的引用 和func2的做法类似 但是函数返回类型不一样 func2不太会产生
// 副作用 但是func4这种情况 在函数返回的过程中不会创建临时对象 main函数中使用ref创建
// result是非常危险的!(使用一个已经析构的对象&ref=mc) -> dangerous!
```
```c++
// 第五种情况 - 同第四种情况 不涉及类型转化的引用返回 不调用构造函数 
myclass & func5(myclass &para){
    // 什么也不调用
    return para;
}
int main(){
    // 调用default构造创建global_mc5
    myclass global_mc5; 
    // 调用copy构造使用para创建result5
    myclass result5=func5(global_mc5);
    // 调用析构函数析构result5 调用析构函数析构global_mc5 析构顺序和入栈相反
    return 0;
}
// 理论输出: construct \n copy \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 with '-fno-elide-constructors' 
// 实际输出: construct \n copy \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 编译器会自动做优化 但是此情况没有copy elision优化的空间 
// 补充: 返回全局变量的引用 和func4做法类似 但是这里返回的是'伪全局'变量的引用
// func4会产生严重的副作用 但是func5使用了'生命期内'的变量构建result -> safe!
```
```c++
// 第六种情况 - myclass &类型转化成myclass 调用copy构造函数
myclass func6(){
    // 调用default构造创建mc
    myclass mc; 
    // no.1: 调用move构造创建一个临时对象 调用析构函数析构局部变量mc
    return mc;
    // no.2: 构建一个局部变量的引用 无需调用任何构造函数
    myclass &ref=mc; 
    // 引用ref如同mc本体 调用copy构造创建一个临时对象 调用析构释放mc
    return ref;
}
int main(){
    // 从myclass类型的临时对象创建一个myclass类型的右值引用 不调用构造函数
    myclass && result6=func6();
    // 调用析构函数释放result6 调用析构函数释放临时对象
    return 0;
}
// 理论输出: no.1: construct \n move \n destruct \n destruct \n
//         no.2: constrcut \n copy \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 with '-fno-elide-constructors' 
// 实际输出: no.1: construct \n destruct \n
//         no.2: construct \n copy \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 编译器会自动做优化 no.1情况move函数可以优化掉 但是
//      no.2情况 由于发生了myclass & -> myclass类型转化 copy不能优化掉 
// 补充: 返回全局变量的引用 和func4做法类似 但是这里返回的是'伪全局'变量的引用
// func4会产生副作用 但是func6使用了'生命期内'的临时变量构建result -> safe!
```
```c++
// 第七种情况 - 返回的形式如同引用定义 不调用任何构造函数
myclass & func7(){
    // 调用default构造创建mc
    myclass mc; 
    // no.1: 相当于 myclass & xxx=mc 无需调用任何构造函数 调用析构释放mc
    return mc;
    // no.2: 和no.1的写法等价 无需调用任何构造函数 调用析构释放mc
    myclass &ref=mc; 
    return ref;
}
int main(){
    // 无需调用任何构造函数 但是这种情况使用了一个已经销毁对象mc的引用
    myclass & result7=func7();
    // main函数中没有需要释放的对象
    return 0;
}
// 理论输出: construct \n destruct \n 
// 解释: x86-64 gcc 10.3 with '-fno-elide-constructors' 
// 实际输出: construct \n destruct \n
// 解释: x86-64 gcc 10.3 编译器没有RVO NRVO优化的空间 
// 补充: 返回局部变量的引用 使用一个已经释放过的变量的ref -> dangerous!
```
```c++
// 第八种情况 - 通常情况下的最优选择 开销小 能修改外部变量 
myclass & func8(myclass &para){
    // 返回'伪全局'变量的引用 不调用任何构造函数
    return para;
}
int main(){
    // 调用default构造创建'伪全局'变量global_mc8
    myclass global_mc8; 
    // 引用的初始化 不涉及对象创建 无需调用任何构造函数 
    myclass &result8=func8(global_mc8);
    // 调用析构释放global_mc8
    return 0;
}
// 理论输出: construct \n destruct \n
// 解释: x86-64 gcc 10.3 with '-fno-elide-constructors' 
// 实际输出: construct \n destruct \n
// 解释: x86-64 gcc 10.3 copy构造含有类型转化 无法省略
// 补充: 返回全局变量的引用 通常这是我们使用引用的目的: 
//      避免在函数传值 返回值的过程中构建无用临时对象 
//      函数对传入值的修改也会反应到外部变量上
```
```c++ 
// 第九种情况 - 使用std::move右值初始化一个左值 优先调用move
int main(){
    // 调用default构造创建mc
    myclass mc;
    // std::move函数用左值创建右值 用move构造将右值资源转移到左值对象上
    myclass result9=std::move(mc);
    // 调用析构释放result 调用析构释放mc
    return 0;
}
// 理论输出: construct \n move \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 with '-fno-elide-constructors' 
// 实际输出: construct \n move \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 move构造函数不能满足RVO条件 move调用不能简化
// 补充: myclass 左值=右值 这是一个定义式 需要调用构造 如果myclass没有
//      定义move构造 则调用copy构造
```
```c++
// 第十种情况 - 使用std::move右值初始化一个右值引用(左值) 调用copy构造
int main(){
    // 调用default构造创建mc
    myclass mc;
    // std:move()从左值构建右值 再使用右值创建右值引用 不调用构造
    myclass &&ref=std::move(mc);
    // 调用copy构造使用右值引用(左值)资源'深复制'给result
    myclass result10=ref;
}
// 理论输出: construct \n copy \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 with '-fno-elide-constructors' 
// 实际输出: construct \n copy \n destruct \n destruct \n
// 解释: x86-64 gcc 10.3 copy一般情况无法简化
// 补充: myclass &&ref=右值 这是一个右值引用的定义 引用的定义不调用构造
//      但是myclass r10=右值引用(左值) 左值初始化左值 调用copy
```

**[总结]** <br>
`1` 能调用move构造函数优先调用move，然后才调用copy构造函数。<br>
`2` 函数构建临时变量用于函数返回时，优先使用move构造函数，省去深复制的资源浪费。<br>
`3` 生命期比较短的变量(编译器分析)初始化其他变量的时候，优先调用move构造函数。<br>
`4` 函数返回值 -> 返回类型，myclass => myclass &类型，不调用构造。<br>
`5` 函数返回值 -> 返回类型，myclass & => myclass类型，使用copy构造。<br>
`6` 不涉及类型转化的引用返回，不涉及构造函数调用。<br>
`7` 如果函数返回形式如同引用定义(同`4`)，不涉及构造函数调用。<br>
`8` myclass &ref=std::move(mc)，std::move右值初始化左值，使用move构造。<br>
`9` myclass &&ref=std::move(mc)，std::move右值初始化右值引用，使用copy构造。<br>

### 左值引用和右值引用
右值引用是c++11引入的新特性，体现了转义语义(move semantics)，和精确传递(perfect forwarding)。它主要目的有两个方面，**消除对象交互时不必要的拷贝，节省储存资源，提高效率**；**能够更简洁明确的调用泛型函数**。

**[关于右值和左值]**：c/c++中所有的表达式和对象，要么是右值，要么是左值。通常定义中，左值是非临时对象，能在多条语句中使用，右值是临时对象，只能够在当前语句中有效。不能以出现在表达式的左边或者右边定义左值右值，左值当然可以出现在右边，右值也可以出现在赋值表达式的左边。
```c++
((i>0? i:j))=1;// 这是一个表达式 等式左边是一个右值(不能接受赋值)
```

**[关于const&和右值引用]**：c++11之前右值不能被引用，只能使用`const&`进行绑定。但是这种情况下，右值无法被修改，这限制了右值的引用场景。实际上，右值可以被修改，因此定义了右值引用，它保留了右值被修改的特性，并且使得引用绑定到右值上，节约了资源。

**[右值引用]**：左值引用的声明符号是&，右值引用的声明符号是&&。右值引用是用来豉汁转义语义(可以将资源从一个对象转移到另一个对象中)的。在构建类的过程中，如果想要实现资源转移(转义语义)，必须为其实现move构造函数。右值引用一般用在函数形参表中，这样函数既可以接受右值也可以接受临时变成右值的左值。
```c++
// myclass &&：接受右值 按引用方式传递；myclass &：接受左值 按引用方式传递
void func(myclass &&, myclass &, myclass);
```

**[左值右值的转化]**：定义std::move函数可以将左值转化成右值使用。注意std::move可以看成是一种强制类型转化，并且std::move不会调用相应类别的move构造函数。
```c++
// 注意 ref是右值引用,但它本身是一个左值 mc是左值 std::move得到相应的右值
myclass &&ref=std::move(mc);
```
### 函数参数使用引用的三种用途
**1** 想要在函数内部不使用指针的方式改变传入参数的值。<br>
**2** 向主调函数返回额外的结果，即除了return语句，可以通过传入引用的方式隐式'返回'想要的数值。<br>
**3** 避免传入较大类型二引起的参数拷贝内存开销。<br>

```c++
// 举例说明第二点
int func(int i, int &ii){// 隐式的通过引用的方式修改外部参数
    i=i*i;
    ii = i*i;
    return i;// 显式地通过return语句返回数值
}
```



## Reference
> cpp primer  <br>
> 关于RVO，NRVO，copy elision的具体规则，详见[这篇博客](/cpp/2021/04/29/post-cpp-copy-elision/)。

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
