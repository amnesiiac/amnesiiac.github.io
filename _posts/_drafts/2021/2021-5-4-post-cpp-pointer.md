---
layout: post
title: "cpp - pointer"
subtitle: 'c++指针相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics\_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-04 11:00
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---

## 1 空悬指针
当一个自动变量的地址被存储在一个生命期长于它的指针时，这个指针被称为空悬指针。即自动变量被销毁，操作指向该自动变量的指针是危险的。

## 2 数组的指针 & 指针的数组
**▶︎ 指针的数组** <br>
指针的数组，数组中的每个元素都是一个指针。

<center><img src="/img/in-post/cpp_img/pointer_1.pdf" width="80%"></center>

**▶︎ 指针的数组的初始化**
```c++
int a, b, c, d;
int *ref_arr[4]={&a, &b, &c, &d};// 写法1 - ref_arr是数组名
(int *) ref_arr[4]={&a, &b, &c, &d};// 写法2 - 和上面的式子等价
```

**▶︎ 数组的指针** <br>
是一个指向数组的指针。

<center><img src="/img/in-post/cpp_img/pointer_2.pdf" width="80%"></center>

**▶︎ 数组的指针初始化:**
```c++
int arr[4]={0, 1, 2, 3};
int (*ptr)[4];
ptr=&arr;// 正确的写法! 
ptr=arr;// 错误的写法! 不同类型之间不能赋值
```

**▶︎ 指针的数组 vs 数组的指针**<br>
分别从类型、数值上进行对于两者进行比较。
```c++
cout<<arr<<endl;// 输出arr的值: 0x7fff8e3022b0
cout<<ptr<<endl;// 输出ptr的值: 0x7fff8e3022b0
cout<<typeid(arr).name()<<endl;// A4_i
cout<<typeid(ptr).name()<<endl;// PA4_i
arr == ptr;// 不同类型的东西 不能直接使用==进行比较!
```
指针的数组和数组的指针分别+1，看看他们存储之间的关系:
```c++
cout<<arr<<"  "<<arr+1<<endl;// 0x7fff8e3022b0  0x7fff8e3022b4
cout<<ptr<<"  "<<ptr+1<<endl;// 0x7fff8e3022b0  0x7fff8e3022c0
```
为什么arr+1，其数值+4呢? 为什么ptr+1，其数值+16? 首先我们需要明确，指向T类型的指针的移动，是以sizeof(T)为单位进行的，具体地，用下面的程序进行解释：
```c++
arr=&arr[0];// 数组名 = 指向数组首元素的指针 => 类型为 int *
arr+1=arr+1*sizeof(arr[0])// 指向int类型的指针的移动，以sizeof(int)为单位进行
ptr=&arr;// 数组的指针 = 指向数组名的指针 => 类型为 int *[4]
ptr+1=ptr+1*sizeof(arr)=ptr+1*sizeof(arr[0])*4=ptr+16// sizeof(int *[4])
```

**▶︎ 指针的数组和数组的指针之间的关系**
```c++
arr=*ptr;// 数组名 = 数组的指针解引用后的数值
ptr=&arr;// 数组的指针解引用 = 取数组名地址
```

**▶︎ c3 programmer course link**
<iframe src="//player.bilibili.com/player.html?aid=29473098&bvid=BV18W411R7zk&cid=80983827&page=83" scrolling="no" border="0" frameborder="yes" framespacing="0" width="100%" height="500px" allowfullscreen="allowfullscreen"> </iframe>

## 3 函数的指针
### 3.1 函数的指针的基本概念
**▶︎ 函数的类型** 函数名不是其类型的一部分，函数的类型只由它的返回值和参数表决定。
函数的指针书写需要注意优先级和结合性，写法同数组的引用以及数组的指针相似。
```c++
int string_cmp(const string &str1, const string &str2){// 函数定义
    return str1.compare(str2);
}
int *ptr_func(const string &, const string &);// 返回类型为int*的函数 不是函数指针
int (*ptr_func)(const string &, const string &);// 函数指针 类型同string_cmp函数
```
函数指针只能指向相同类型的函数，对于类型不同的函数，函数指针无法指向它。
```c++
int sum(int, int);// 这个函数无法使用ptr_func函数指针来指向
int (*ptr_sum)(int, int);// ptr_sum函数指针可以指向sum函数
```

**▶︎ 参数表中含有省略号的函数指针** 下面的函数参数表虽然只相差ellipsis符号，但是不能共用相同的函数指针。
```c++
int printf1(const char *, ...);// 含有ellipsis
int printf2(const char *);// 没有ellipsis
int (*ptr1)(const char *, ...);// printf1类型函数指针
int (*ptr2)(const char *);// printf2类型函数指针
```
更多关于ellipsis以及函数使用可变数量、类型参数的知识，详见[这篇文章](/cpp/2021/05/05/post-cpp-uncertain-parameters/)。

**▶︎ 函数指针的初始化和赋值** 不带下标的数组名会被解释成指向数组首元素的指针，同理，当一个函数名没有被调用操作符修饰时，会被解释成指向该类型函数的指针。将取地址操作符用在函数上也能产生指向该类函数的指针。
```c++
int string_cmp(const string &, const string &);// 函数声明
string_cmp;// 被解释成int (*)(const string &, const string &)类型的函数指针
&string_cmp;// 同被解释成int (*)(const string &, const string &)类型的函数指针
```
指向函数的指针可以按照如下的方式进行初始化：
```c++
int (*ptr1)(const string &, const string &)=string;// 初始化方式1
int (*ptr2)(const string &, const string &)=&string;// 初始化方式2
ptr1=string;  ptr2=&string;  ptr3=ptr1;// 三种赋值方式
```
在指向函数的指针之间不存在隐式类型转化，只有初始化和赋值左右两边类型完全相同时，才能完成相应操作，否则编译器报错。
```c++
int (*ptr1)(const string &, const string &)=0;// 赋予0值 不指向任何函数
int (*ptr2)(const string &, const string &)=nullptr;// 同上 不指向任何函数
```

**▶︎ 通过函数指针调用函数** 函数调用可以直接通过函数名的方式进行，也可以通过函数指针进行。
```c++
int string_cmp(const string &, const string &);// 函数声明
int (*ptr)(const string &, const string &)=string_cmp;// 函数指针初始化
string_cmp(xxx, xxx);// 通过函数名直接调用string_cmp函数
ptr(xxx, xxx);// 函数指针调用string_cmp方式1 - 隐式函数指针调用
(*ptr)(xxx, xxx);// 函数指针调用string_cmp方式2 - 显式函数指针调用
```
虽然函数指针调用方式1和调用方式2是等价的，但是推荐使用第二种显式方式进行使用，方便阅读出ptr是一个函数指针。

**▶︎ 函数指针的数组** 即数组中的所有元素都是指向函数的指针，具体见下面代码：
```c++
int string_cmp(const string &, const string &);// 声明一个函数
int (*ptr_func)(const string &, const string &);// 声明一个函数指针
int (*ptr_func_arr[10])(const string &, const string &);// 声明函数指针的数组
```
第三行声明了一个含有10个类型为`int (*)(const string &, const string &)`元素的数组，这个数组的名字为`ptr_func_arr`。这种声明方式太复杂了，也很不直观，下面通过`typedef`关键字给出一种简化的声明：
```c++
typedef int (*T)(const string &, const string &);// typedef声明方式
int (*ptr_func_arr[10])(const string &, const string &);// 原始的声明
T ptr_func_arr[10];// 一个等价的声明 - 利用typedef进行简化
```
上面的第三行的形式非常直观，符合常规数组的定义模式。

**▶︎ 函数指针的数组的初始化** 和常规数组的初始化方式一样，以函数指针类型作为基本元素的数组也可以用初始化列表的方式进行初始化，具体代码如下：
```c++
int (*func_ptr1)(const string &, const string &);// func_ptr1
int (*func_ptr2)(const string &, const string &);// func_ptr2
typedef int (*T)(const string &, const string &);// define arr
T func_ptr_arr[2]={func_ptr1, func_ptr2};// init by 初始化列表
```

**▶︎ 指向函数指针的数组的指针** 听起来很复杂，但是只不过是简单知识的多层堆砌罢了。结合数组的指针以及函数的指针的数组两部分知识：
```c++
(T *) new_func_ptr_arr[2];// 指针的数组 每个元素都是T*类型
T (*func_ptr_arr_ptr)[2];// 数组的指针 每个元素都是T类型
```

**▶︎ 初始化方式** 分别介绍指向函数指针的数组的指针的初始化方式，以及，指向函数指针的数组名的初始化方式。LOL。
```c++
*func_ptr_arr_ptr = &func_ptr_arr;// 数组的指针进行初始化 - 同数组指针初始化
func_ptr_arr=&func_ptr_arr[0];// 数组名初始化
```

**▶︎ 利用(复杂形式的)函数指针调用函数** 变量命名规则: func\_ptr\_arr=指向函数的_指针的_数组；func\_ptr\_arr\_ptr=指向函数的_指针的_数组的_指针_。显然，前者是一个数组名，后者是一个指向数组的指针。利用上述两个指针调用**最底层**的函数的方式详见下面的代码：
```c++
(func_ptr_arr[0])(string str1, string str2);// 调用方式1 - 隐式
(*(func_ptr_arr[0]))(string str1, string str2);// 调用方式2 - 显式
((*func_ptr_arr_ptr)[0])(string str1, string str2);// 调用方式3 - 隐式
(*((*func_ptr_arr_ptr)[0]))(string str1, string str2);// 调用方式4 - 显式
```
其中显式和隐式具体区别见上文**通过函数指针调用函数**部分，

### 3.2 函数指针的用法
上面列举了各种函数指针，函数指针的数组，指向函数指针的数组的指针等等，这一部分重点关注函数指针的使用。

**▶︎ 函数指针作为参数传递** 这样做的好处是，可以在函数内部，通过函数指针调用相应的函数；并且，可以通过改变传给函数的函数指针，实现内部不同函数的调用。
```c++
int sort(string *, string *, `函数指针`);// 排序函数定义模型
int sort(string *, string *, int (*)(const string &, const string &));
typedef int (*T)(const string &, const string &);// 简化函数指针类型
int sort(string *, string *, T);// 简化后的排序函数定义
int sort(string *, string *, T=mycompare);// 提供缺省函数指针
```
当我们大部分情况下，都调用相同的比较函数进行比较时，可以将对应的函数指针传递给声明，作为缺省的函数指针参数，方便在默认的情况下进行调用。

**▶︎ 函数参数列表中不能含有函数类型** 函数中参数列表中的函数类型声明将被自动转化成指向该类型函数的指针类型。
```c++
typedef int func_type(const string &, const string &);// 函数类型
typedef int (*T)(const string &, const string &);// 函数指针类型
void sort(string *, string *, func_type);// func_type函数类型被自动转化
void sort(string *, string *, T);// 编译器自动转化成函数指针类型
```
第三行和第四行的声明是等价的，编译器自动的将声明中的参数类型转化。

**▶︎ 函数返回类型中不能是函数类型** 函数指针类型可以作为函数的返回值，但是，如果返回一个函数类型，那么将会产生编译错误。
```c++
typedef int func_type(const string &, const string &);
func_type sb_func(xxx);// 编译错误! 无法返回函数类型
```

**▶︎ 指向extern "C"类型函数的指针** 可以声明一个函数指针，它指向一个用其他语言便携的函数。
```c++
extern "C" void (*func_ptr)(const string &, const string &);// 声明
```
此时，func\_ptr指向了一个用c语言编写的函数，通过func\_ptr函数指针调用函数时，只能调用c语言编写的函数。<br>
注意，c语言编写的函数和c++语言编写的函数，在使用函数指针调用时，不能混用，否则产生编译错误! 虽然有时，当c语言函数和c++语言函数特性写法相同时，编译器能够接受不同语言函数版本作为扩展，但是尽可能不要这样做。

**▶︎ extern "C"具有'延展性'** 使用外部链接声明的函数指针声明，不仅要求指针实例化的函数是外部链接类型，函数参数列表中声明的函数也被extern "c"作用到:
```c++
extern "C" void f1( void (*func_ptr)(int) );// 声明了f1以及参数表中的函数指针
```
此时，f1是一个c语言类型的函数，而且，func\_ptr函数指针锁指向的函数也是一个c语言函数。<br>
那么如何在c++类型的函数中，声明一个c类型的函数指针呢？答案是使用`typedef`。
```c++
extern "C" typedef void (*extern_func_ptr)(int);
void f1(extern_func_ptr);// 此时f1是c++类型 参数类型是c类型
```
上面的式子中，extern\_func\_ptr是一个c语言类型的函数的指针，千万要注意`extern`和`typedef`关键字的写法顺序哦：`typedef void(xxx)`是一个整体，`extern "C"`是一个外部修饰。

**▶︎ 指向重载函数的指针** 可以声明一个指向一组重载函数的指针：
```c++
extern void overload(unsigned int);
extern void overload(float);
void (*func_ptr)(unsigned int) = &overload;// 函数指针指向第一个函数
void (*func_ptr1)(float) = &overload;// 函数指针指向第二个函数
```
函数类型由返回类型以及函数参数表决定，和函数名无关。重载函数具有相同的函数名，参数表不同，根据参数表可以确定函数指针具体指向哪一个重载的函数。


## 4 类的成员函数的指针 
函数指针在上面的内容中已经有所介绍，这一部分主要介绍类的成员函数指针。

### 4.1 为什么需要定义函数指针 
因为：将函数定义成指针后，可以更加方便的进行函数重复调用。参考下面的例子进行理解：
```c++
// ---------- user.h ----------
class Screen{
public:
    inline Screen & up();// cursor向上移动一个screen元素
    inline Screen & down();// cursor向下移动一个screen元素
    Screen & repeat(char, int);// no.1 定义一个重复光标移动函数
    Screen & repeat(Action, int);// no.2 定义一个重复光标移动函数
private:
    int row; int col;
};
// ---------- user.cpp 第一种方法 ----------
Screen & repeat(char op, int times){
    switch(op){
        case up:
            // call screen::down() n times - 通过直接调用函数的方式
            for(int i=0; i<times; ++i){
                up();            
            } break;
        case down:
            // call screen::up() n times - 通过直接调用函数的方式
            for(int i=0; i<times; ++i){
                down();
            } break;
        default: ;
    }
}
// ---------- user.cpp 第二种方法 ----------
tyepdef Screen & (Screen::*Action)();// 定义一个类成员函数指针
Screen & repeat(Action op, int times){
    for(int i=0; i<times; ++i){
        (this->*op)();// this对象调用特定成员函数
    }
}
```
第一种方法定义的repeat通过接受一个char类型的标识符，并在repeat函数内部通过switch-case语句对于up()、down()操作进行分别调用。第二种方法定义的repeat通过接受一个函数指针参数，来实现，通过直接传递函数的方式进行功能切换，相比第一种方式，更加简明、清晰。

函数指针可以作为其他函数的参数进行传递，也可以作为其他函数的返回类型。使用函数指针而不是直接调用特定函数的意义在于：函数指针代表了一类函数操作，对于类型完全相同，仅仅名字不同的一组函数，函数指针作为参数能够增加主调函数的复用性。

### 4.2 类成员函数的指针的定义&初始化
```c++
int & (*ptr)()=0;// 普通函数指针的定义
Screen & (Screen::*s_ptr1)()=0;// 类成员函数的定义
Screen & (Screen::*s_ptr2)()=nullptr;// 类成员函数的定义
```
注意，类成员函数的定义中，需要对`*`操作符通过类名进行修饰限制：`Screen::*`，表示这是一个Screen类的指针。类的函数指针和普通函数指针之间不能互相赋值，即使它们的返回类型、参数表都完全相同。
```c++
Screen & (*ptr)()=0;// 普通函数指针
s_ptr1 = s_ptr2;// 正确 同一类的成员函数指针可以互相赋值
ptr = s_ptr1;// 错误 类的成员函数指针和普通函数指针不能直接赋值
```
使用typedef技术可以帮助简化类成员函数指针形式上的繁琐，并且能够实现一定程度上的信息隐藏。
```c++
typedef Screen &(Screen::*Action)()=0;// typedef用于简化复杂类型表示
Action s_ptr2 = Screen::up();// 初始化一个类成员函数指针
Action s_ptr3 = s_ptr2;// 定义并初始化了类成员函数指针s_ptr3
```

### 4.3 类成员函数指针的使用
**[1]** 通过下面的例子可以对类成员函数指针的初始化、及其typedef简写方式进行了解，并掌握通过类的对象、对象的引用、对象的指针进行类成员函数指针间接调用的方法。
```c++
using namespace std;
class screen{
    public:
        inline int up(){return num;};
    private:
        int num;
};
// no.1 定义一个类的成员函数指针 全局对象含有默认值nullptr
int (screen::*func_ptr)();
// no.2 定义一个类的成员函数指针 指定初始值screen::up
int (screen::*func_ptr)() = &screen::up;
// 定义一个类的成员函数指针的<简写>
typedef int (screen::*action)();
action act=func_ptr;
// init screen obj & ref_obj & ptr_obj
screen s; screen &ref_s = s; screen *ptr_s = &s;
int main(){
    // no.1 类的对象、类的对象的引用、类的对象的指针直接调用成员函数
    s.up(); ref_s.up(); ptr_s->up();
    // no.2 类的对象、对象的引用、对象的指针通过成员函数指针间接调用成员函数 
    (s.*func_ptr)();
    (ref_s.*func_ptr)();
    (ptr_s->*func_ptr)();
    // no.3 类的对象、对象的引用、对象的指针通过成员函数指针的<简写>调用成员函数
    (s.*act)();
    (ref_s.*act)();
    (ptr_s->*act)();
    return 0;
}
```

**[2]** 通过下面的例子可以了解：类的成员函数指针作为函数参数进行传递的方式，及其调用的技巧：<br>
一种常规的函数指针作为参数进行调用的方法：
```c++
Screen &(Screen::*func_ptr)();
typedef Screen &(Screen::*Action)();
Screen & move(Action);// 声明：move函数接受一个函数指针作为参数
move(&Screen::up);// 调用：直接将函数地址传入
```
通过构建类的成员函数指针表的方式提供给一个调用的索引，并借助枚举类型对于索引中被传入的函数进行选择：
```c++
// 构建类的成员函数指针表 - 每个元素都是一个函数指针
Action func_ptr_list[]={&Screen::up(), &Screen::down()...};
// 通过设置函数参数为枚举类型来实现：成员函数选择性传递
enum cursor_movement={UP, DOWN};// up=0, down=1
Screen & move(cursor_movement);// move declare in class 
Screen & Screen::move(cursor_movement cm){// move def
    (this->*func_ptr_list[cm])();// call 
    return *this;
}
```

**补充一点：**类成员函数指针可用于指定函数参数缺省实参。
```c++
Screen & act(Screen &, Action=&Screen::up);// 在函数声明中指定缺省实参
```

### 4.4 static类成员的指针
static类成员的指针和上一节的普通类的成员函数指针不同，它不具备this指针。static类成员属于类的全局对象or函数。static类成员(包括对象&函数)的指针属于普通指针。
参考下面的例子可以简略的对static类数据成员、函数成员的指针的初始化、用法进行了解：
```c++
// ---------- user.h ----------
class Account{
public:
    static void raiseinterest(int incr);
    static double interest(){return _interestrate;}
    double amount(){return _amount;}
private:
    static double _interestrate;
    double _amount;
    string _owner;
};
// ---------- non-inline.h ----------
void Account::raiseinterst(int incr){
    _interestrate+=incr;
}
// ---------- user.h ----------
int main(){
    // no.1a static类数据成员指针的 <定义> & <初始化>
    Account acc;
    double Account::* acc_ptr = &acc._interestrate;// 类型错误
    double *normal_ptr = &Account::_interestrate;// 类型正确
    // no.1b static类数据成员指针可以像普通指针一样<解引用>
    double dailyrate = *normal_ptr/365;
    // no.2a static类函数成员指针的 <定义> & <初始化>
    void (*func_ptr)(int) = &Account::raiseinterest;// right!!!
    void (Account::*func_ptr)(int) = &Account::raiseinterest;// wrong!!!
    // no.2b non-static类成员函数指针的 <> & <初始化>
    double (*func_ptr2)() = &Account::interest;// wrong!!!
    double (Account::*func_ptr2) = &Account::interest;// right!!!
    // no.2b static类函数成员指针的 <解引用>
    ...// 按照正确的类型进行应用即可 略
}
```


## Reference
> c++ primer 3ed p318 <br>
> https://www.cnblogs.com/liushui-sky/p/10025732.html <br>
> c++ primer 3rd chapter 13.6 p535 (类的成员函数的指针) <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/xxx.pdf" width="100%"></center>`
