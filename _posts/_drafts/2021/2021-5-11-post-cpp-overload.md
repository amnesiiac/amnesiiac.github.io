---
layout: post
title: "cpp - overload"
subtitle: 'overload重载函数相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-11 10:20
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## 为什么需要重载函数机制
c语言中函数名字必须唯一，在c++中增加了函数重载机制，使得可以声明一组相同名字的函数。函数重载机制可以简述为：想要调用一个函数，通过相同的函数名进行索引，并根据参数表类型通过`重载函数解析机制`从一组重载的函数中选择最优执行。<br>
**重载函数适用于定义一组大体功能相近，并且通过参数表决定具体执行方案的程序逻辑框架中应用。**
```c++
// 使用不同函数名进行区分
void int_max(int, int);
void double_max(double, double);
// 使用重载机制简化 - 虽然我认为这个例子声明称函数模版更适合
void max(int, int);
void max(double, double);
```


## 函数重载规范
**[1]** 重载函数中的参数个数或者类型不同，名字相同，则这组函数为重载函数。
```c++
void max(int, int);
void max(double, double);
```
**[2]** 重载函数返回类型和参数表完全匹配，名字相同，则这组函数为同一个函数的重复声明。
```c++
void max(int, int);
void max(int i, int j);// 参数表和第一个完全匹配
```
**[3]** 重载函数除了缺省实参形式不同，其他都相同，函数名字相同，则这组函数为同一个函数的重复声明。
```c++
void max(int, int);
void max(int, int j=1);// 声明第二个参数为缺省参数(含有默认值)
```
**[4]** 重载函数除了返回类型不同，其他都相同，函数名字相同，则这组函数为同一个函数的**错误**重复声明。
```c++
void max(int, int);
int max(int, int);// 这是一个错误的重复声明 - 导致编译错误
```
**[5]** 重载函数部分使用了typedef关键字将类型名字改写，其他都相同，函数名字相同，则这组函数为同一个函数重复声明。
```c++
typedef int INT;
void max(int, int);
void max(INT, INT);// 是一个重复的声明 不构成函数重载关系
```
**[6]** 重载函数的参数表中，除了const、volatile关键字不同，其他都相同，函数名字相同，则这组函数为同一函数的重复声明。但是const、volatile如果应用在指针参数上，则声明了不同的重载函数。
```c++
// no.1 值类型
void max(int, int);
void max(const int, int);// 重复的声明1
void max(volatile int, int);// 重复的声明2
// no.2 指针
void max(int *, int *);
void max(const int *, int *);// 重载声明1
void max(volatile int *, int *);// 重载声明2
// no.3 引用
void max(int &, int &);
void max(const int &, int &);// 重载声明1
void max(volatile int &, int &);// 重载声明2
```
**[6]补充** 类内的成员函数重载时，如果const修饰this指针，则它可以和没有const修饰的版本构成重载关系。此时，类对象的常量性决定了调用关系。
```c++
class Screen{
public:
    char get(int x, int y);// 重载声明1
    char get(int x, int y) const;// 重载声明2
};
int main(){
    Screen s; const Screen c_s;// 常量类对象调用const版本 非常量调用普通版本
    s.get(1, 2);// 调用重载声明1
    c_s.get(1, 2);// 调用重载声明2
}
```
但是类内的构造函数和析构函数的调用中，对象的常量性不能决定调用构造函数的调用关系：常量对象和非常量对象都可以调用同一构造函数，准确的说，当对象构造完成之后，它的const属性才被建立起来。常量对象和非常量对象都可以调用同一析构函数，准确的说，当对象调用析构的时刻，它的const属性已经被拿掉。简而言之：对象的const属性只在从对象构造完成到对象析构之前存在。

**[7]** 重载函数和函数局部作用域之间的关系。重载函数的所有声明应当在逻辑上一个域进行声明，不同域之间的函数不能构成重载关系，只能构成覆盖、隐藏的关系。 
```c++
void print(const string &);// overload 1
void print(double);// overload 2
void func(int val){
    // 将外部声明引入func局部域中 - 这个函数屏蔽了全局域中的2个print声明
    extern void print(int);
    print("value");// 试图调用全局域中的函数是错误的 编译器只能看到局部的print
    print(val);// 正确 局部域中print(int)可见
}
```
**[8]** 重载函数和名字空间作用域之间的关系。重载函数不能跨越不同的名字空间定义的域。
```c++
namespace space0{
    void print(const string &);// overload 1
    void print(double);// overload 2
}
namespace space1{
    void print(int);// 这个函数没有和前面两个print构成重载关系
}
```
**[8.1]** using声明完成跨域重载函数声明。使用using声明可以将名字空间作用域中的函数在其他地方使用。需要注意的是：`1`using声明引入的函数就好像在该声明出现的地方被声明一样。`2` using声明引入的函数和在声明域中同名的函数参数表相同则声明冲突。`3` using声明只能一次将一组重载函数都引入，试图通过using将重载函数中的某一个函数引入是不合理，不可实现的。
```c++
namespace space0{
    int max(int, int);
    int max(double, double);
    void print(const string &);
    void print(double);
}
using space0::max;// 正确 在全局域中引入名字空间space0中的函数max
using space0::print(double);// 错误! using声明引入函数不能带参数表
void print(double);
using space0::print();// 错误! 重复的print声明
void func(){
    max(2, 3); max(2.2, 3.3);// max重载声明在func()局部域中可见
}
```
**[8.2]** using指示符完成跨域重载函数声明。需要主要的是：using指示符使名字空间的成员就像在名字空间之外的地方被声明的一样。
```c++
namespace space0{
    extern void print(int);
    extern void print(double);
}
extern void print(const string &);
using namespace space0;// 将名字空间space0中所有的声明都引入到当前域中
using namespace space1;//  将其他名字空间中的声明引入当前域中
void func(int val){
    // 此处可以使用space0以及space1共同构建的重载函数集
    print("value");  print(val);
}
```
**[9]** extern "C"和重载函数。关于这部分的内容，详见[这篇博客](/cpp/2021/04/22/post-cpp-extern/)中的第三部分：用法三。


## 何时不要使用重载函数
如果不同的函数名所提供的信息可使程序更易于理解的话，则再用重载函数就没有什么好处了。
```c++
// 重载函数的写法 - 模糊
void num(int);// 接受一个int类型数据
void num(int, int);// 计算两个int类型数据之和
// 不使用重载函数的写法 - 更加清晰
void get_num(int);
void add_num(int ,int);
```


## 函数重载和函数模版的区别和联系
**[1] 重载函数** <br>
使用相同的函数名，不同的参数表(不同的参数数量或不同的参数类型)，不同的函数内部执行逻辑。<br>
**[2] 函数模版** <br> 在具体实例化后，使用相同的函数名，不同的参数表(相同的参数数量设置，只是参数的类型不同)，完全相同的函数内部执行逻辑(只有执行的类型不同)。<br>
**[3] 结论** <br>
重载函数 - 适用于参数表不同，具体操作也不同的一组功能相近或者逻辑相近的函数。 <br>
函数模版 - 适用于参数表中只有类型不同，具体操作完全一致的一组类型决定的函数集合。


## 重载解析的三个步骤
重载解析(function overload resolution)是把函数调用和一组重载函数中的某一个相匹配的过程；还包括：决定是否能够找到有效匹配的重载函数，决定在匹配的过程中是否存在二义性乃至能够成功完成重载函数调用。
```c++
void f();
void f(int);
void f(double, double=3.4);
void f(char *, char *);
int main(){
    f(5.6);
    return 0;
}
```
**[1] 确定函数调用时需要考虑的重载函数集合；确定函数调用者的参数表类型**。<br>
选出的被考虑的重载函数集合又称为候选函数(candidate function)。**候选函数满足两个条件**：候选函数和调用函数同名；候选函数声明在调用点所在的域中可见or实参类型所在的名字空间中的同名函数。上述例子中重载函数集合中的4个函数都是候选函数。上述例子中调用者的参数表类型为一个double类型的实参。

**[2] 从候选函数集合中选出可行函数，所有可行函数都满足函数调用者的需求**。<br>
在第一步中选出的候选函数中进行选择，通过和函数调用者的参数表类型进行匹配，**将所有能够完成匹配(能够完美匹配或者通过类型转化的方式、或者为多余的参数提供了缺省值进行调用的)称为可行函数(viable function)**。如果候选函数中没有能够成功被调用者使用，则重载函数调用是错误的，属于无匹配情况(no match situation)。上述例子中可行函数有两个：一个通过类型转化的方式进行调用，另一个通过为对于的参数提供了默认的实参进行调用。

**[3] 在可行函数中，选择最佳可行函数**。<br>
在这一步中，所有的可行函数都可以成功的被调用者调用，在选出最佳匹配的函数过程中，需要为所有不能够完美匹配的类型转化、多余参数提供默认值等操作设置rank，以区分出优劣；从可行函数选出最佳函数的过程中，如果不能够选出唯一的最佳函数，则称重载函数的调用是二义性的(ambiguity)。这种分辨转化的优劣依赖两条规则：`1` 应用在实参上的转换不比调用其他可行函数所需的转换差；`2` 在某些实参上的转换要比其他可行函数对该参数的转换好。根据这两条规则：上述例子中void func(double, double=3.4)是一个更好的转化。

### 重载解析中第一步 Find候选函数
**[1] 候选函数的选取范围**
- 在调用点可见的同名函数。
```c++
void func();
void func(int);
void func(double);
func(5.6);// 调用func(double)
```
- 实参类型所在的名字空间中声明的命名函数，即使它们并不可见，但是仍然作为候选函数。
```c++
namespace NS{
    class C{...};
    void takeC(C&);// candidate function
}
NS::C cobj;
int main(){
    takeC(cobj);// 参数为名字空间NS中定义的类C的对象
}
```

**[2] 候选函数的选取注意事项**
- 嵌套域内部的候选函数隐藏了外部同名函数，因此候选函数只考虑未被隐藏的部分。
```c++
char* func(int);// no.1 域外的同名函数被隐藏 不是候选函数
void test_func(){
    char* func(double);// no.2 候选函数 
    char* func(char);// no.3 候选函数
    func(3);// 调用func(double)
}
```
- 使用using声明通过增加可见域的方式扩大候选函数搜索范围。
```c++
// 案例一 using声明 && using指示符
namespace NS{
    int max(int, int);// no.1
    double max(double, double);// no.2 
}
char max(char, char);// no.3
using namespace NS::max;// 使用using声明将名字空间中的max函数引入到全局域中(可见)
using namespace NS;// 使用using指示符将名字空间中所有东西都引入
int main(){
    max(1, 2);// call no.1
    max(1.1, 2.2);// call no.2
    max('f', 'k');// call no.3
}
```
```c++
// 案例二 using声明
namespace NS{
    int max(int, int);// no.1
    double max(double, double);// no.2 
}
char max(char, char);// no.3 -> 被屏蔽
int main(){
    using namespace NS::max;// 使用using声明将名字空间中的max函数引入局部域中
    max(1, 2);// call no.1
    max(1.1, 2.2);// call no.2
    max('f', 'k');// call no.1 char->int 标准转化中整型提升
}
```
```c++
// 案例三 using指示符 - 和using声明有所区别
// [使得名字空间中的名字好像它们在名字空间之外、在定义名字空间的位置被声明一样]
namespace NS{
    int max(int, int);// no.1
    double max(double, double);// no.2 
}
char max(char, char);// no.3 -> 没有被屏蔽
int main(){
    using namespace NS;// 使用using指示符将名字空间中所有东西引入
    max(1, 2);// call no.1
    max(1.1, 2.2);// call no.2
    max('f', 'k');// call no.3
}
```

### 重载解析中第二步 候选函数的参数全部匹配=可行函数
**[1] 精确匹配(exact match)**。<br>
实参和函数参数类型精确匹配。精确匹配包括：左值到右值的转换、从数组到指针的转换、从函数到指针的转换、限定修饰转换。
```c++
// 重载声明
void print(unsigned int);
void print(const char*);
void print(char);
// 分别精确匹配上述声明
unsigned int a;
print(a);// 匹配unsigned int
print("a");// 匹配const char *
print('a');// 匹配char
```
**[2] 和一个类型转化相匹配(type conversion)**。<br>
实参不能直接和参数类型相匹配，但是通过类型转化之后能够和参数类型匹配。类型转化可以分成3种类型：提升(promotion)、标准转换(standard conversion)和用户定义的转换(user-defined conversions)。其中，用户定义的转化包括：单参数非显式构造函数、类定义中的类型转化函数。<br>
三种类型转化优先级关系为：提升优先级 \> 标准转化 \> 用户定义的转化。
```c++
void func(char);// 一个重载的声明
func(0);// 通过类型转化将int转化成char
```
**[3] 无匹配(no match)**。<br>
实参不能直接匹配、也不能通过类型转化和参数类型进行匹配。
```c++
class my_int{/* */}
my_int i;
print(i);// 错误! 在重载函数中不能找到匹配的类型
```


#### 精确匹配
**[1] 实参与函数参数的精确匹配** <br>
```c++
// no.1 普通精确匹配
int max(int, int);// no.1
double max(doble, double);// no.2
max(2, 3);// 精确匹配no.1
max(2.2, 3.3);// 精确匹配no.2
```
```c++
// no.2 枚举类型的精确匹配
// 枚举类型定义的枚举值 以及 枚举类型创建的对象 与枚举类型本身精确匹配
enum Tokens{INLINE=128, VIRTUAL=129};// 枚举类型1
Tokens curTok = INLINE;// 枚举类型1的对象
enum Stat{Fail, Pass};// 枚举类型2
extern void func(Tokens);
extern void func(Stat);
func(curTok);// 枚举类型创建的对象 -> 枚举类型
func(Pass);// 枚举类型的枚举值 -> 枚举类型
```

**[2] 左值到右值的转化** <br>
```c++
// 左值 - 可寻址的对象 通过对象的地址可以获得它的值
// 右值 - 是一个表达式 表示一个值 不可被寻址
int calc(int);
int main(){
    int ival, res;
    ival = 5;// ival:左值  5:右值
    res = calc(ival);// res:左值  用于存放calc返回的临时变量:右值
}
```
```c++
// no.1 某些情况下 只需要左值表达式提供数值
int i, j;
int main(){
     int local = i+j;// 首先将i、j左值表达式中的值抽离出来(左值->右值) 再记进行加法
}
```
```c++
// no.2 当函数需要一个按值传递的实参 但是传入的实参是一个左值时 左值->右值
string color("purple");
void print(string);
print(color);// 将左值color转化成右值 再按值传递 
```
```c++
// no.3 当函数需要一个按引用传递的实参 不需要 左值->右值 转化
void print(list<int>&);// 函数按引用传参
list<int> mylist(20);
print(mylist);// 不发生 左值->右值 转化
```

**[3] 数组到指针的转化** <br>
```c++
// 数组名作为参数总是自动转化成数据类型的指针
int arr[3];
void sum_up(int *);
sum_up(arr);// 隐式发生 数组->指针
```

**[4] 函数到指针的转化** <br>
```c++
// 函数名作为参数总是自动转化成函数类型的指针 
int lexico_compare(const string&, const string&);
typedef int (*func_ptr)(const string&, const string&);// 定义函数指针类型
void sort(string*, string*, func_ptr);// sort函数类型声明
string str[10];
sort(str, str+(str/str[0])-1, lexico_compare);// 自动发生 函数名->函数指针 的转化
```

**[5] 限定修饰转化 - 只影响指针(const & volatile)** <br>
```c++
// no.1 用于将const或volatile加到指针上：
int a[5]={1, 2, 3, 4, 5};
bool is_equal(const int*, const int*);// 函数声明 接受const int*类型参数
int *ptr_a=a;
int func(int *ptr_tmp){
    if(is_equal(ptr_a, ptr_tmp)){// 隐式通过限定修饰转化 int* -> const int*
        ...
    }
}
```
```c++
// no.2 当参数类型不是指针时 不能发生限定修饰转化 
extern void func(const int);
int i;
func(i);// 不发生限定修饰转化 -> [实参本身即函数参数的精确匹配]
```
```c++
// no.3 当const或volatile被用于修饰指针指向的对象而非指针本身 不会发生限定修饰转化
extern void func(int* const);
extern int *ptr;
func(ptr);// 不会发生限定修饰转化 -> [实参本身即函数参数的精确匹配]
```

**[6] 只需左值转化(lvalue transformation)的精确匹配要比需要限定修饰转化的精确匹配要好。** <br>
左值转化包括：[2]左值->右值、[3]数组->数组指针、[4]函数->函数指针。

**[7] 可以使用显式强制类型转化来指定精确转化的类型(可用来打破可行函数调用的二义性)**
```c++
extern void func(int);
extern void func(void*);
func(0xffbc);// 调用func(int) oxffbc是int的16进制文字常量
func(reinterpret_cast<void*>(oxffbc));// 使用显式强制转化可以调用func(void*)
```

#### 经过类型转化之后再匹配
**[1] 提升(promotion)** <br>
- char、unsigned char、short类型实参提升为int类型。如果机器上int长度\>short长度，则将unsigned short提升为int，否则提升为unsigned int类型。在任何机器上，int类型长度都大于char、unsigned char，因此它们默认提升为int类型
```c++
// char && unsigned char && short -> int
extern void print(unsigned int);
extern void print(int);
extern void print(char);
unsigned char ch;
print(ch);// 调用print(int) 执行提升unsigned char -> int
```
- float类型实参提升为double类型
- 枚举类型实参(枚举常量or枚举类型)被提升成下列第一个能够表示其所有枚举常量的类型：int、unsigned int、long、unsigned long
```c++
// enum -> int && unsigned int && long && unsigned long
// no.1 普通的情况
enum Stat{Fail, Pass};
extern void func(int);
extern void func(char);
func(Pass);// 枚举常量->int
// no.2 稍微特殊的情况 
// 枚举值为0、1、2 均可使用char类型表示 编译器会将 枚举类型->char类型
enum CHAR{zero, one, two};
// 0x80000000不能用char、int进行表示 编译器会将 枚举类型->unsigned int类型
enum UNSIGNED_INT{a, b, c=0x80000000};
// Test
string format(int);
string format(unsigned int);
format(CHAR);// -> int
format(UNSIGNED_INT);// -> unsigned int
```
- 布尔类型实参被提升为int类型

**[2] 标准转化(standard conversion)** <br>
- ⚠️  整型类型转化：任何整型类型 or 枚举类型->其他整型类型(不包含提升部分涉及的整型转化)
- ⚠️  浮点类型转化：任何浮点类型->其他浮点类型(不包含提升部分涉及的浮点类型转化)
- ⚠️  浮点类型<->整型类型：相互转化
```c++
extern void func(double);
int i; func(i);// int->double
```
- 指针转化：整型0值->指针类型的转化 or 任何类型的指针-\>void\*类型的转化
```c++
extern void func(void*);
int i; func(&i);// int*->void*
```
- 布尔类型转化：任何整型类型、枚举类型、浮点类型、指针类型-\>bool类型的转化

**关于标准转化的注意事项**：<br>
- ⚠️  符号表示相应转化可能有潜在危险，这是因为转化的目标类型不能表示源类型的所有值，这是被称作标准转化而不是提升转化的原因。浮点数表示形式可理解为：$$floatnum = \frac{1}{2} + \frac{1}{4} + \frac{1}{8} + ...$$逼近分数序列构成，表示形式本身可能损失部分精度。关于浮点数表示的详细内容，可参考深入理解计算机系统chapter2。
```c++
// float类型不能表示int类型的所有精度
extern void func(float);
int i; func(i);// int->float 可能会丢失精度
```
- 所有标准转化都是等价的，且标准转化的优先级和源类型到目标类型的相近程度无关。
```c++
extern void func(long);
extern void func(float);
func(3.14);// wrong! 3.14默认为double类型 double->long? or double->float?
// solve - 可以使用强制类型转化
func(static_cast<long>(3.14));// ok 调用func(long)
// solve - 可以使用float文字常量后缀
func(3.14F);// ok 调用func(float)
```
- 枚举类型不能直接完成指针转化 
```c++
extern void func(int*);
func(0L);// ok! long->int*
func(0x00);// ok! hex->int* 
enum ZERO{zero=0};
func(zero);// wrong! 枚举类型不是整型 不能完成指针转化
```
- 常量0值优先调用func(int)
```c++
extern void func(int);
extern void func(void*);
func(0);// 优先调用func(int) 无转化优先级>标准转化优先级
```
- 只有数组指针才能转化成void\*，函数指针不能转化成void\*
```c++
typedef int (*func_ptr)();// 函数指针
extern func_ptr arr[10];// 函数指针数组 
extern void reset(void*);
reset(arr[0]);// wrong! 函数指针int (*)()类型不能转化成void*类型
```

#### 引用机制对于参数类型匹配机制的影响
**[1] 传入的实参类型为引用** <br>
- 实参的类型不可能是引用类型，将一个引用类型的实参传递相当于传递了引用所指的对象。
```c++
int i; int &ref=i;
void print(int);
print(i);// 调用print(int)
print(ref);// 同上 调用print(int) 传入的实参类型为int而不是int&
```
- 传入实参类型如果是引用，则类型转化(提升or标准转化)仍然对引用所指类型生效。
```c++
int i; int &ref=i;
void print(double);
print(i);// ok! int->double
print(ref);// ok! int->double 引用所指类型为int 相当于对引用所指类型进行类型转化
```

**[2] 函数参数类型为引用** <br>
- 实参是函数引用参数的精确匹配。
```c++
void swap(int&, int&);// 函数参数为引用
int i,j; swap(i,j);// ok! 精确匹配
```
- 实参不能初始化函数引用参数。
```c++
// no.1 案例一
int i;
extern void func(double&);// 函数参数为引用
func(i);// wrong! 标准转化int->double 得到的结果是临时值 临时值不能用于初始化引用
```
```c++
// no.2 案例二
class B;
extern void takeB(B&);
extern B offerB();
takeB(offerB());// wrong! 临时对象不能用于初始化引用
```
```c++
// no.3 案例三
extern void print(int);
extern void print(int&);
int i; int &ref=i;
print(i);// wrong! 调用二义性 int->int int->int& 都是精确匹配
print(ref);// wrong! 调用二义性 ref相当于i 原因同上
print(86);// ok! 86->int 精确匹配 86不是非const引用的初始值 
```
(a) 关于为什么临时对象不能用于初始化引用类型：c++标准中不允许修改临时对象，进而临时对象本身具有const属性，因此临时对象只能用于初始化常量引用。<br>
(b) 关于产生临时对象的几种场景：临时对象通常产生于如下4种情况，类型转化、函数参数按值传递、函数按值返回、对象定义。<br>
(c) 对象定义的三种常见形式：
```c++
Panda huanhuan(200);// 一定不会产生临时对象
Panda huanhuan = Panda(200);// [可能]产生临时对象 -> 大概率被优化掉
Panda mengmeng(200); Panda huanhuan = mengmeng;// [可能]产生临时对象
```
**[总结]** 函数参数为引用，如果实参是一个有效初始值，则肯定是精确匹配，否则不匹配。

### 重载解析中第三步 Find the Best可行函数(将转化序列划分等级)
**[1] 最佳可行函数中的参数匹配符合下面两个条件：**
- 用在传入实参上的转化不比调用其他可行函数需要的转化更差
- 用在某些传入实参上的转化比其他可行函数的需要的转化更好 

**[2] 找到最佳可行函数的过程可以抽象为：**寻找优先级最高的转化序列(converison sequence)的过程。转化序列的优先级是转化序列的最坏等级。一种粗略的划分优先级的方式是：精确匹配\>提升\>标准转化。

**[3] 标准转化序列(涉及多次类型转化时的转化模版)。** <br>
0或1个左值转化 ➤  0或1个提升 ➤  0或1个标准转化 ➤  限定修饰转化 <br>
其中0表示不转化，1表示至多进行一次该类型转化。另外，关于用户定义的转化序列及其在类继承层次关系中的应用可参考[\[Link\]](/cpp/2021/06/17/post-cpp-inheritance-2/#确定最佳可行函数)。

**[4] 关于标准转化序列的一些更加细致的规则案例**
- 如何根据标准转化序列判断重载函数调用优先级
```c++
namespace NS{
    int max(int, int);// no.1 按值传递
    double max(double, double);// no.2 按值传递
}
using NS::max;
int main(){
    char c1, c2;
    max(c1, c2);// 调用max(int, int) 提升转化优先级>标准转化优先级
}
// no.1 转换序列 左值->右值(从char中抽取值) => char->int(提升转化)
// no.2 转换序列 左值->右值(从char中抽取值) => char->double(标准转化)
```
- 一个二义性调用的例子
```c++
int i, j;
extern long func(long, long);// no.1 按值传递
extern double func(double, double);// no.2 按值传递
void test(){
    func(i, j);// wrong! 二义性调用 没有最佳匹配 两个转化优先级相同
}
// no.1/2 转化序列 左值->右值(从int中抽取值) => int->long/double(标准转化)
```
- 限定转化优先级属于精确匹配，但也比无需限定修饰转化要差
```c++
// 案例一 限定修饰符影响按值传递的参数匹配等级
void reset(int*);// no.1 按值传递 
void reset(const int*);// no.2 按值传递
int *ptr;
int main(){
    reset(ptr);// 调用reset(int*) 两个序列优先级相同 无需限定转化的序列更好
}
// no.1/2 转化序列 左值->右值 + int*->int* 或 int*->const int*(均为精确匹配)
```
```c++
// 案例二 限定修饰符影响按引用传递的参数匹配等级
void manip(vector<int>&);// no.1 按引用传递
void manip(const vector<int>&);// no.2 按引用传递
extern vector<int> vec;
vector<int> func();
int main(){
    manip(vec);// 调用no.1
    manip(func());// 调用no.2
}
// manip(vec): 和no.1/2函数都是精确匹配 但是无需限定转化的匹配优先级更高
// manip(func()): 函数返回临时值(const) 只有一个可行函数no.2 - 常量引用
```
- 当函数有1个以上的实参，则全部实参的标准转化序列都需要被考虑
```c++
extern void func(char*, int);// no.1 按值传递
extern void func(int, int);// no.2 按值传递
int main(){
    func(0, 'a');// 综上 选择no.1作为最佳可行函数
}
// 0->char*(标准转化中的指针转化)  0->int(精确匹配) no.1更好
// 两个可行函数都进行 'a'->int 转化 no.1/2等价
```
多参数匹配依据的原则：**(1)** 全部实参的标准转化序列都需要被考虑。**(2)** 在某些参数上转化序列不比其他可行函数中的差。**(3)** 在某些参数上的转化序列要好于其他可行函数。

**[5] 关于缺省实参对于最佳可行函数选择的影响**
缺省实参可以进一步扩大可行函数搜寻的范围：可以允许可行函数的参数个数多于函数调用参数表中的参数个数。
```c++
extern void func(int);// no.1 按值传递
extern void func(long, int=0);// no.2 按值传递 缺省实参
int main(){
    func(2L);// 精确匹配func(long, 0) 多余参数有缺省实参
    func(0, 0);// int->long(标准转化中的整型转化) 匹配func(long, 0)
    func(0);// 精确匹配匹配func(int)
    func(3.14);// 二义性调用 double->int? or double->long?(均为标准转化)
}
```


# Reference
> 《c++ primer》3rd chapter 9

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
