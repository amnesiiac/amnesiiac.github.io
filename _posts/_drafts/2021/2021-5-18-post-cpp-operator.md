---
layout: post
title: "cpp - operator"
subtitle: 'operator进行符号重载相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-18 19:50
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## 1 赋值 & 逻辑 & 运算 操作符重载
### 1.1 operator=
```c++
class String{
public:
    // 参数:[const char*] 而不是 [char*] -> 能够允许 String str="hello" 赋值
    // 返回:[String&] -> 能够让重载的操作符进行 [连续赋值] 
    String & operator=(const char *);// override '=' in member func
    String & operator=(const String &);// override above op=
private:
    int _size;// num of char elements
    char * _string;// low-level storage: char array 
};
String & String::operator=(const char *ctr){
    if(ctr==nullptr){// exception handle
        _size=0;
        delete [] _string;
        _string=nullptr;
    }
    else{
        _size=strlen(ctr);
        delete [] _string;// free inner memory
        _string = new char* [_size+1];// c风格字符串后面有'\n'
        strcpy(_string, ctr);
    }
}
```


### 1.2 operator==
为类的对象重载逻辑等于操作符。如果定义的类需要具有比较(== or >= or <=)逻辑关系，则应当为其重载逻辑操作符。<br>
**逻辑等于操作符重载注意事项：** <br> 
**[1]** 重载逻辑等于操作符可以重载为成员函数，也可以重载为友元，也可以通过普通函数访问类开放api完成。推荐使用成员函数重载做法实现。<br>
下面给出三种方法的声明、实现程序(自定义String类型的逻辑==操作符重载)：
```c++
// ---------- user.h ----------
class String{
public:
    String(const char*, size);
    size_t size(){return _size;}// api for _size
    string & get_string(){return _string;}// api for _string
    bool operator==(const String&);// op== member func
    friend bool operator==(const String&, const String&);// op== friend func
private:
    string _string;
    size_t _size;
};
// Method-1 [普通函数]通过调用类内开发的api的方式完成类对象的比较
bool operator==(const String &str1, const String &str2){
    if(str1.size() != str2.size()){// access by api 
        return false;
    }// strcmp函数通过c_str()接口获取底层c风格字符串
    return strcmp(str1.get_string().c_str(), str2.get_string().c_str())? \
        false : true;
}
// Method-2 [友元函数]通过类内声明友元+直接调用类的私有成员的方式完成对象比较
bool operator==(const String& str1, const String &str2){
    if(str1.size() != str2.size()){
        return false;
    }
    return strcmp(str1.c_str(), str2.c_str())? false:true;
}
// ---------- define.h ----------
#include "user.h"
// Method-3 [成员函数]通过直接调用类的私有成员的方式完成对象比较
bool String::operator==(const String &str){
    if(str.size() != _size){
        return false;
    }
    return strcmp(str.c_str(), _string.c_str())? false:true
}
// ---------- user.cpp ----------
int main(){
    const char *ctr1="hello";
    const char *ctr2="world";
    String str1(ctr1, 6);
    String str2(ctr2, 6);
    operator==(str1, str2);// method-1 普通函数+类接口api -> str1==str2
    operator==(str1, str2);// method-2 友元函数 -> str1==str2
    str1.operator==(str2);// method-3 成员函数 -> str1==str2
}
```
**比较上述三种重载op==的方法：**<br>
Method-1普通函数虽然调用了类内api函数，但是定义在类内的函数默认是inline的，所以执行的效率和Method-2友元函数近似，但是友元函数破坏了类成员的封装性，因此成员函数较好。Method-3成员函数直接访问内部数据的方式进行对象比较，通过this指针，不破坏封装性，效率高。<br>
从使用形式上进行比较，三种方法的应用形式都为str1==str2，符合直观认知。


### 1.3 operator ++ & \-\-
为类的对象重载自增自减操作符：如果想要定义的类类型的对象能够支持自增自减运算符(如：构建一个用于指向顺序存储元素的指针类)，则应当定义operator++()、operator\-\-()、operator++(int)、operator\-\-(int)。<br>
**重载自增自减操作符注意事项：** <br>
**[1]** 应当直接将重载的操作符函数定义为成员函数，因为直接使用\*this对于这个操作符的定义很方便! <br>
**[2]** 也可以将自增自减操作符定义成友元函数，因此需要将\*this变成友元函数的参数显式传递给operator函数。
```c++
// --------- user.h -----------
#include <cstddef> // for size_t
class ScreenPtr{// 重载了++/--操作符 用于指向一个screen数组中的元素
public:
    // constructor
    ScreenPtr(Screen &s, size_t arrsize=0):size(arrsize),offset(0),ptr(&s){}
    // overload prefix operator
    Screen & operator++();
    Screen & operator--();
    // overload postfix operator
    Screen & operator++(int);
    Screen & operator--(int);
    // friend overload prefix operator
    friend Screen & operator++(ScreenPtr &);
    friend Screen & operator--(ScreenPtr &);
    // friend overload postfix operator
    friend Screen & operator--(ScreenPtr &, int);
    friend Screen & operator--(ScreenPtr &, int);
private:
    size_t size;// arr size
    size_t offset;// record ptr offset in arr before operate ++/==
    Screen *ptr; 
};
// --------- definition.h -----------
#include "user.h"
Screen & ScreenPtr::operator++(){// prefix ++
    if(size==0){
        std::cerr<<"cannot increment pointer by single object!"<<std::endl;
        return *ptr;
    }
    if(offset>=size-1){// 对ptr+1指向的元素进行操作 需保证ptr指向arr中的元素
        std::cerr<<"already at the last position of array"<<std::endl;
        return *ptr;
    }
    ++offset; return *++ptr;// update offset & return
}
Screen & ScreenPtr::operator++(int){// posfix ++
    if(size==0){
        std::cerr<<"cannot increment pointer by single object!"<<std::endl;
        return *ptr;
    }
    if(offset==size){// 对ptr指向的元素进行操作 需保证ptr指向arr中的元素
        std::cerr<<"already at one past position of array"<<std::endl;
        return *ptr;
    }
    ++offset; return *ptr++;
}
Screen & ScreenPtr::operator--(){// prefix --
    if(size==0){
        std::cerr<<"cannot decrement pointer by single object!"<<std::endl;
        return *ptr;
    }
    if(offset<=0){// 对ptr-1指向元素进行操作 需保证ptr-1指向arr中的元素
        std::cerr<<"already at first position of array"<<std::endl;
        return *ptr;
    }
    --offset; return *--ptr;
}
Screen & ScreenPtr::operator--(int){// postfix --
    if(size==0){
        std::cerr<<"cannot decrement pointer to single object!"<<std::endl;
        return *ptr;
    }
    if(offset==-1){// 对ptr指向的元素进行操作 需保证ptr能够指向arr中的元素
        std::cerr<<"already at one before position of array"<<std::endl;
        return *ptr;
    }
    --offset; return *ptr--;
}
// ---------- user.cpp ----------
const int arrsize=10;
Screen *p_arr = new Screen[arrsize];
ScreenPtr ptr_arr(p_arr, arrsize);// int -> size_t
for(int i=0; i<arrsize; i++){
    printScreen(ptr_arr++);// [postfix++的隐式调用] 利用ptr对相应的元素进行操作
}
```
声明和定义后置++/\-\-的成员函数或者友元函数时，需要额外在参数表中添加int作为标识。编译器会为这个int类型参数提供一个缺省值，所以在声明、定义、调用的过程中，不用用户显式给出它的数值。另外，通过显式调用后置++/\-\-操作符的方式如下：
```c++
ScreenPtr ptr_arr(p_arr, arrsize);
ptr_arr.operator++();// 调用前置++操作符函数
ptr_arr.operator++(1024);// 显式地给后置++操作符函数提供一个数值
```

## 2 输入输出流操作符重载(用于将类的对象的输入输出)
### 2.1 operator>> & operator<<
为类的对象重载输入输出操作符：如果想要自定义的类的对象用于输入或者在其他地方(console、显示器)进行输出显示，则应当为其重载输入输出运算符。<br>
**输入输出操作符的重载注意事项** <br>
**[1]** 重载输入输出操作符应当使用友元函数来完成。使用成员函数也能重载，但是使用形式不符合直观认知。<br>
```c++
// ---------- user.h ----------
class Screen{
public:
    ...
    // member
    istream & operator>>(istream &);
    ostream & operator<<(ostream &)const;
    // friend
    friend istream & operator>>(istream &, Screen &);
    friend ostream & operator<<(ostream &, const Screen &);
private:
    size_t _size;
    string _string;
}
// member func op== def
istream & Screen::operator>>(istream &in){...}// no idea how to def
ostream & Screen::operator<<(istream &out){
    out<<"<"<<_height<<","<<_width<<">"<<endl; 
    out<<_screen;
    return out;
}
// friend func op== def 
istream & operator>>(istream &in, Screen &s){...}// no idea how to def
ostream & operator<<(ostream &out, const Screen &s){
    out<<"<"<<s._height<<","<<s._width<<">"<<endl; 
    out<<s._screen;
    return out;
}
// ---------- user.cpp ----------
// member func --> weird
s.operator>>(cin);  s>>cin;
s.operator<<(cout);  s<<cout;
// friend func --> prevalent
operator>>(cin, s);  cin>>s;
operator<<(cout, s);  cout>>s;
```
根据上述user.cpp中的使用方式来看，对于自定义类的输入输出流运算符重载只能采用友元函数的方式，其输入输出流的使用方式复合直观认知，而成员函数重载的方式非常诡异。关于友元函数相关知识，可参考这篇[博客](/cpp/2021/05/17/post-cpp-friend/)。


## 3下标访问操作符重载(行为类似数组的类)
### 3.1 operator[]
为类的对象重载下标访问操作符：一般情况下，为**表示一种抽象数据类型的容器并且允许获取其单个元素的类**定义operator[]。<br>
**下标操作符的重载注意事项：** <br>
**[1]** 重载下标操作符必须被定义为该类的成员函数。<br>
**[2]** 重载下标操作符必须能够出现在等式的两边：为了能够在等式左边出现(接受赋值)，操作符需要返回一个左值，类似前面`op=`、`op>>`、`op<<`的做法，可以通过返回引用来实现。
```c++
#include <cassert>
class String{
    public:
        // 返回[char&] 可以通过 str.op[x]=y 的形式修改str中的数据
        inline char& operator[](size_t idx)const;// no.1 返回左值
        // 返回[char] 则 str.op[x]=y 的形式只能修改返回的char的拷贝 不能修改str本身
        inline char operator[](size_t idx)const;// no.2 返回左值 但是没意义
    };
    inline char& String::operator[](size_t idx)const{
        assert(idx>=0 && idx<_size);// boundary check
        return _string[idx];
}
```
上述下标访问操作符返回的是一个左值，这使得它能够完成下面的赋值操作。
```c++
String str("melongogogo");
str[0]='M';// 重载的下标操作符可以在等式左边被赋值
```


## 4 函数调用操作符重载(类本身定义了一类操作)
### 4.1 operator()
为类的对象重载函数调用操作符：如果一个类类型被定义用来标识一类操作时，则可以为它定义operator()。<br>
**函数调用操作符重载注意事项：** <br>
**[1]** 重载的操作符必须被定义成成员函数。<br>
**[2]** 重载的操作符()参数列表可以接受任何能够传递给函数的参数；重载操作符的返回值可以为任何能够被返回的类型。
```c++
// ----------- user.h ----------
class AbsInt{// AbsInt被用来定义一类操作: 将int转化成abs(int)
public:
    // no.1 int-abs func
    inline int operator()(int);
    // no.2 template func: 实例化之后的成员函数和int-abs构成重载关系 
    template<typename T> inline T operator()(T);
};
// ----------- headerfile.h ----------
#include "user.h"
inline int AbsInt::operator()(int val){
    return val<0? -val:val;
}
template<typename T> inline T AbsInt::operator()(T val){
    return val<0? -val:val;// must override op< & op- for type T
}
// ----------- user.cpp ----------
AbsInt func; int a=3; func(a);
AbsInt funcT; double b=-3.2; funcT(b);
```
上面对op()的使用中，func是一个AbsInt类操作的对象，通过它能够完成将传入数据取绝对值的操作。成员函数给出了一个重载函数模版的abs版本，通过这个重载函数实现对double类型数据进行abs操作。另外一种做法是将AbsInt类写成模版类(更加直观)。


## 5 成员访问操作符重载(行为类似指针的类)
### 5.1 operator->
为类的对象重载成员访问操作符箭头：如果想要定义一个类类型的对象具有类似内置指针一样的作用，一般这样的对象具有智能指针类似的行为，则为这个类定义operator->。<br>
**成员访问操作符重载注意事项：** <br>
**[1]** 重载的操作符必须被定义成成员函数。
```c++
// ---------- user.h ----------
class ScreenPtr{// 定义指向screen类的一个指针
    public:
        // constructor
        ScreenPtr(Screen & s):ptr(&s){}
        // overload operators to make ScreenPtr morelike pointers
        Screen& operator*(){return *ptr;}// dereference op overload
        Screen* operator->(){return ptr;}// member access op arrow overload
    private:
        screen *ptr;// ptr member
};
// ---------- user.cpp ----------
ScreenPtr p;// wrong! 没有为ScreenPtr定义默认构造函数
Screen myscreen(4, 4);
ScreenPtr p(myscreen);// right! call the constructor defined
*p.action();// right! op*() defined! *p是screen类对象 
p->move(2, 3);// right! op->() defined! 指向screen类的指针
```


## 6 New & Delete (为类的对象添加动态内存管理操作)
### 6.1 operator new & delete
如果没有为类类型定义new&delete操作符，则存储在自由存储区的对象的分配和释放，通过调用c++标准库中的定义的全局操作符new()和delete()来进行。我们可以通过为类类型定义new&delete操作符来完成自定的内存分配、释放操作。

**重载new()&delete()操作符注意事项：**<br>
**[1]** 在调用特定类的new()或delete()操作符时，编译器优先使用类内定义的new()或delete()操作符，如果类内没有定义new()或delete()则调用全局操作符。<br>
**[2]** 操作符new()的返回类型必须是void\*类型，并且带有一个size_t类型的参数。<br>
**[3]** 操作符delete()的返回类型必须是void，并且第一个参数类型是void\*。<br>
**[4]** delete可以接受两个参数，且第二个参数类型必须是预定义类型size_t(cstddef)，并且这个参数被编译器用void*指针指向的对象在内存中占用字节的数量进行初始化(**用于存储分配对象占用内存**)。当delete操作符被继承类使用时，第二个参数比较重要，详见[继承类中的delete如何使用]。<br>
**[5]** 如果使用全局new操作符分配内存、构建对象，则应当使用全局delete操作符析构对象、返回内存。<br>
**[6]** new()和delete()自动被设置成静态函数，new和delete是类的静态成员，这一点无需程序员在定义时显式声明。static member没有this指针，它只能访问static成员。解析：new和delete操作符要么运用在对象构建之前，要么运用在对象析构之后，因此，不能给予它访问非static数据成员的权限。<br>

**new()&delete()操作符的使用方法**
```c++
class Screen{
public: 
    void* operator new(size_t);// declare of op new
    void operator delete(void*);// declare of op delete
}
Screen *ptr = new Screen;// usage of new
delete ptr;// usage of delete
```
上述表达式在没有为Screen自定义new()&delete()操作符时也可以正常使用，只不过调用的是全局的new&delete操作符。另外，程序员可以显式通过全局域访问`::`调用全局new&delete操作符：
```c++
// usage of global new() & delete() 
Screen *ptr = ::new Screen;
::delete ptr;
```
另外，delete函数还可以接受两个参数，第二个参数必须是预定义类型size_t。size_t类型的参数被编译器用void*指针指向的对象在内存中占用字节数量进行初始化：
```c++
class Screen{// operator delete
    void operator delete(void *ptr, size_t);// size_t = sizeof(*ptr)
}
```
new()&delete()操作符的隐式、显式调用的方式如下(展示了new/delete expression和new/delete operator之间的关系)：
```c++
// implicit new = new expression = new表达式
Screen *ptr = new Screen(10, 20);
// explicit new
Screen *ptr = Screen::operator new(sizeof(Screen));// operator new:用于分配内存
Screen::Screen(ptr, 10, 20);// construct Screen obj
```
```c++
// implicit delete = delete expression = delete表达式
delete ptr;
// explicit delete
Screen::~Screen(ptr);// destruct Screen obj
Screen::operator delete(ptr, sizeof(*ptr));// op delete ptr:用于释放内存
```

### 6.2 array operator new[] & delete[]
上一部分中的operator new和operator delete是用于当个类型对象管理，这一部分中介绍的数组操作符用来管理特定类型的对象数组。如果一个类型没有定义自己的数组操作符new[]和delete[]，则默认使用全局数组操作符进行类数组的管理。

**重载new[]和delete[]数组操作符注意事项：**<br>
**[1]** 在调用类型的new[]或delete[]暑促操作符进行类数组管理时，如果类内没有定义new[]或delete[]数组操作符，则使用全局定义的new[]或delete[]数组操作符。<br>
**[2]** new[]数组操作符的返回类型必须是void\*，并且第一个参数是size_t类型。<br>
**[3]** delete[]数组操作符返回类型必须是void，并且第一个参数是void\*类型。<br>
**[4]** 使用new[]数组操作符创建类的数组时，需要保证该类已经定义了缺省构造函数，如果该类为定义缺省构造函数，则相应new表达式是错误的。<br>
**[5]** 使用delete[]数组操作符销毁类的数组时，要使用delete[]而不是delete，前者对于创建的每个类的对象依次调用析构函数，并在最后返回占用的内存；后者只对数组首对象调用析构函数，释放的内存数量可能是正确的。关于为什么对于类的数组调用delete也能正确释放内存的原因见本节后续的分析。
**[6]** delete[]数组操作符可以接受两个参数，第二个参数必须是size_t(cstddef)，这个参数被编译器用来存储**数组所有元素**所需内存大小。

```c++
// ---------- user.h ----------
class Screen{
public:
    void* operator new[](size_t);// declare op new[]
    void operator delete[](void*);// declare op delete[]
};
// ---------- user.cpp ----------
Screen *ptr = new Screen[10];// 在内存上用默认构造函数创建了Screen对象数组
Screen *ptr = ::new Screen[10];// 调用全局new[]创建Screen对象数组
delete[] ptr;// 析构ptr指向的数组对象 并返回相应内存
::delete[] ptr;// 调用全局delete[]完成对象析构和内存释放
```
**[关于注意事项[5]的补充]** <br>
**\[Question\] \[delete\[\]和delete同样接受一个void\*类型的指针，为什么delete[]能够'智能地'析构正确数量的类对象，并且能够返回正确数量的占用的内存呢\]** <br>
**[Answer]** 首先在调用new[]的时候，进行内存的分配，这个时候，编译器可以向[含有第二个参数的new[]数组操作符传递分配数组的对象占用内存大小]；如果没有显式定义new[]数组操作符的第二个参数，则编译器自动开辟一个size_t大小的内存用于记录数组对象占用内存大小，这种方法称为cookie。在构建对象的过程中，同样编译器写一个日志，用于记录构建的数组类对象的数量。因此，在调用delete[]进行对象析构和内存销毁时，能够正确的析构对象释放内存。<br>
**\[Question\] \[为什么对于类对象的数组调用delete只析构首元素但返回正确数量的内存\]** <br>
**\[Answer\]** 根据上述分析可知：如果对于一个new[]构建的类的数组调用delete进行回收的话，由于没有[]符号，编译器不知道需要释放的是一个数组，从而只对数组首元素调用析构函数；在返回内存的过程中，由于new操作符已经将占用内存数量写入编译器cookie中，因此可以返回正确数量的内存。

### 6.3 placement operator new() & operator delete() 
本小节介绍关于new和delete操作符重载的用法。c++中原生支持placement-new，但是没有提供placement-delete(关于placement-new详见[博客](/cpp/2021/05/07/post-cpp-new-delete/))。定位new只在已经分配的内存上构建类对象，相当于将new操作符中的第二步单独提出来用；至于为什么c++没有显示提供定位delete见本小节最后的补充内容。我们这小节为Screen类重载了定位new和定位delete。

**重载定位new()的注意事项：** <br>
**[1]** 任何一个类成员new操作符的第一个参数都必须是size_t类型。至于其他重载参数类型，如下面的Screen*，则可以通过调用new定位操作符时，显式进行初始化。
```c++
// ---------- user.h ----------
class Screen{
public:
    void* operator new(size_t, void*) throw();// 标准定位new
    void* operator new(size_t, Screen*);// 重载定位new 
};
// ---------- user.cpp ----------
Screen *ptr1;
Screen *ptr2 = new Screen;// 调用new操作符默认版本
Screen *ptr3 = new (ptr1) Screen;// 在ptr1分配的内存中创建对象并返回指向它的指针ptr3
```
上述第三行调用了Screen类重载的定位new运算符，在特定的已经构造好的内存中创建Screen对象，并返回指向这个对象的指针。具体地，第三行代码做了如下三件事：
[1] 显式调用类中重载的定位new操作符Screen::operator new(size_t, Screen*)。
[2] 调用类的缺省构造函数初始化这个对象。
[3] 将这个对象的地址传给ptr3指针作为返回。

**[如果不为Screen类定义一个和上述重载new操作符相匹配的delete会出现什么问题？如果重载的new操作符在执行上述第二步出现异常，那么对应的内存该如何释放？怎么避免内存泄漏？]** <br>
使用原生的delete操作符显然不能满足我们的要求，因为编译器找不到申请内存时对应的delete。此时，我们需要定义一个placement delete来完成对placement new构建的对象、内存的管理工作(虽然称为定位delete但是做的事情和delete一致)。

**定义定位delete()的注意事项** <br>
**[1]** 任何一个类成员的操作符第一个参数都应该返回void*，返回类型必须是void。
```c++
class Screen{
public:
    void* operator new(size_t, void*) throw();// 标准定位new
    void* operator new(size_t, Screen*);// 重载定位new
    void operator delete(void*, Screen*);// 定义定位delete对应定位new
}
```
有了上面定义的定位delete函数，在定位new第二步构建对象抛出异常的时候，编译器优先考虑函数类型相匹配的delete。根据操作符参数表规则，new操作符第一个参数总是size_t类型，返回类型总是void*，delete操作符第一个参数总是void*，返回类型总是void类型，因此在new to delete参数表匹配的过程总是从第二个参数开始。至此，问题中定位new第二步抛出异常后，可以调用定位delete进行特定的处理解决内存问题。

**[为什么c++没有提供原生的placement-delete支持]** <br>
**为什么需要定义placement-new：**new expression同时做了两件事：分配对象需要的内存，在内存上构建对象。有的时候，我们想要隔离内存分配和对象创建操作(如：一次性分配一大块内存，再指定内存上的对象构造操作以减少实时分配构造造成的内存碎片化)，因此定义placement-new是有很大实际意义的。<br>
**为什么不必须定义placement-delete：** 反观和new对应的操作符delete，同样完成了两个操作：析构构造的对象，释放占用的内存。那么placement-delete应该被定义成：在指定的内存空间上释放对象，这是不析构函数的功能么？如果我们想要借助placement同时管理对象和内存，那么重载delete函数也是好选择。当然，为了能够和我们重载的placement-new进行配对，我们也可以自定义相应的placement-delete与之对应。<br>

> 因为析构函数可以直接调用。<br>
> 构造函数没办法直接通过指向对象的指针进行调用。<br>
```c++
Screen *ptr = new Screen;
ptr->Screen::Screen();// 错误! 没有这种调用方式!!!
```
> 而析构函数可以直接通过指向对象的指针调用。<br>
```c++
Screen *ptr = new Screen;
ptr->Screen::~Screen();// 正确! 通过Screen对象的指针调用析构函数
```


## 7 Conversion Function(为类添加隐式类型转化功能)
### 7.1 operator type()
本小节介绍类的类型转化操作符的重载方法。如果希望定义的类型能够被编译器隐式地转化成特定的类型，则应该为这个类重载类型转化函数。<br>
**重载类型转化操作符注意事项** <br>
**[1]** 类型转化操作符形如operator type()，其中type可以是内置类型、类类型、typedef类型；不允许type表示**数组**或者**函数类型**！<br>
**[2]** 类型转化函数被许被定义为成员函数，并且他的声明不能指定返回类型和参数表。
下面用自定义的SmallInt为例对于重载方式进行分析：<br>
**[3]** 强制类型转化如static_cast\<int\>会调用相应的类型转化函数int()。<br>
**[4]** 重载类型转化时，对于从类类型到类的私有数据类型的转化需要谨慎，如果私有数据类型是指针类型，则需要注意用户代码通过类型转化对私有数据成员修改的问题。(潜在破坏类的封装性 - 可以 通过定义转化为常量指针来避免 or 通过改为转化成其他的非指针类型来避免)。<br>
**[5]** 使用operator type()完成类型转化之后，Token类型能够应用在任何int类型能够应用的地方。<br>
**[6]** 对于需要多层类型转化的场景：使用用户自定义的转化之后，只允许使用类型的标准转化序列(standard conversion sequences)；如果为了达到目标类型，必须应用第二个用户定义的转化，则编译器不会应用**任何隐式**的转化。违反规则将导致编译错误。<br>
**[7]** 在类类型和要转化的数据类型没有一一对应的逻辑关系时，尽量不要定义转化函数(产生模糊的转化结果)。

**[一种too young too simple的做法]** 为了实现类类型到其他类型的转化，先提供一种非常容易想到的思路：我们可以通过友元函数来实现SmallInt和其他算数类型的运算：
```c++
// ---------- user.h ----------
class SmallInt{
    friend operator+(const SmallInt &, const int &);// version 1
    friend operator+(const int &, const SmallInt &);// version 2
    friend operator-(const SmallInt &, const int &);// verison 3
    friend operator-(const int &, const SmallInt &);// version 4
public:
    SmallInt(int val):value(val){}; 
    operator+(const SmallInt &);
    operator-(const SmallInt &);
private:
    int value;
};
// ---------- user.cpp ----------
SmallInt snum(3);
int result = snum + 3.14;// 调用version 2重载的运算符函数进行运算
int result = 3.14 - snum;// 调用version 4重载的运算符函数进行运算
```
**[一种标准的做法]** 上述友元函数的方式能够解决类型转化的问题，如果我们需要为SmallInt类和所有其他内置类型的运算，按照上述方式显然是费力不讨好的。c++本身为这种类型转化提供了一种机制：能够直接定义类型之间的转化操作，我们只需要为自定义类型SmallInt重载这个转化操作即可；详见下面的代码：
```c++
// ---------- user.cpp ----------
class SmallInt{
public:
    SmallInt(int val):value(val){};
    operator int(){return value;}// conversion func SmallInt => int
private:
    int value;
};
```
上述重载的int()转换函数能够使得自定义类型SmallInt能够用在任何能使用int类型的地方：
```c++
SmallInt snum(3);
snum + 3.14// invoke 2 implicit conversion
```
上面代码中完成了两次类型转化，一次是SmallInt通过重载的int()转化函数变成int类型，一次是int类型通过算数提升转化变成double类型。

类型转化函数(conversion function)是一种特殊类型的类的成员函数，它定义了一个由类设计者定义的类型转化方式。这个转化可以是自定义类型到内置类型的转化，也可以是类类型到类类型之间的转化(如下所示)：
```c++
#include "user.h"
typedef char *tname;
class Token{
public: 
    Token(char *, int);
    // 定义了三个隐式类型转化函数
    operator SmallInt(){return val;}// Token -> SmallInt
    operator tname(){return name;}// Token -> tname
    operator int(){return val;}// 借助SmallInt类型到int类型的隐式转化
private:
    SmallInt val;
    char *name;
};
```
**[重载类的类型转化函数可能由副作用]** 以上述Token类为例，为Token类型定义到char*类型的转化可能会带来副作用：
```c++
typedef char* ctr;
typedef const char* c_ctr;
class Token{
public:
    ...
    operator ctr(){return name;}// no.1 token -> char*
    operator c_ctr(){return name;}// no.2 token -> const char*
    operator string(){return name2;}// no.3 token -> string 
private:
    SmallInt val;
    char *name;
    string name2;
};
```
我们不希望用于能够访问、修改上面代码中Token的私有数据成员，但是如果我们在Token类中定义了到私有成员类型char*的转化，隐式应用这种转化可能会修改私有成员name(通常是不合理的)：
```c++
Token tk("function", 78);
char *tokName = tok;// 通过隐式转化将Token类型的tok转化成char*类型
*tokName = 'P';// Token类型对象tok的私有数据成员变成 'Punction'
```
上述代码中，由于Token类定义了**[类类型->私有数据成员类型的隐式转化]**(如no.1方案所示)，因此，当我们在用户代码隐式使用这种转化的时候，则可以通过用户数据直接对私有成员进行修改，简而言之：**这种做法破坏了类的封装性**。一种解决这类问题的办法是：定义到常量类型的转化(如上述no.2方案所示)。另一种解决此类问题的办法是：定义到非指针类型的转化(如上述no.3方案所示)。

**[类型转化可以同时同地发生多次]** 在需要类型转化的场景中，类型转化允许发生多次转化以满足目标表达式的要求：
```c++
Token tok("constant", 44);
extern void calc(double);
calc(tok);// 在参数传值的过程中同时同地发生两次类型转化 Token->int->double
```
**[某些场景下不要为类定义类型转化函数]** 并不是所有的类型都适合定义类型转化函数，如果在类类型和转化的函数类型之间没有一对一的严格逻辑匹配，则尽量不要定义这种类型转化，如下所示：
```c++
class Date{
public:
    operator int(){return ???};// 定义了一个从Date类型到int类型的转化
private:
    int month, day, year;// 到底应该返回哪个数据成员?
};
```
上述情况下，无论返回哪个数据成员都是片面的，在Data类类型和要转化的int数据类型没有一一对应的逻辑关系时，尽量不要定义转化函数。

### 7.2 使用构造函数作为转化函数 
在类的构造函数中，凡是只带一个参数的构造函数，都定义了一组从参数类型到类类型的隐式转化。

**[内置类型转化成用户定义类型的例子]**
```c++
class SmallInt{
public:
    SmallInt(int i);// 定义了一组从int类型到SnallInt类型的隐式转化
};
int i;
extern void calc(SmallInt);
calc(i);// 隐式转化 int -> SmallInt
```
上述通过calc(i)调用隐式转化的过程可以用c++伪代码进行理解：
```c++
{// 指出了用于类型转构建的临时对象的生命期
    SmallInt tmp = SmallInt(i);// 通过SmallInt构造函数构造一个临时对象
    calc(tmp);// 再将类型转化好的临时对象传入函数作为参数
}
```
**[用户定义类型转化成用户定义类型的例子]**
```c++
class Number{
    Number(const SmallInt &);// 定义了一种从SmallInt到Number类型的转化
};
SmallInt si(87);
extern void func(Number);
func(si);// SmallInt可以被用在任何需要Number类型数据的地方
```
**[关于类型转化序列的问题]** 如果需要，编译器会在调用构造执行用户定义的转化**之前**，在实参上应用标准转化序列：
```c++
double d_num;
extern void calc(SmallInt);
calc(d_num);// 编译器首先进行标准转化序列double->int 然后int->SmallInt
```
**[单参数构造函数隐式自动转化带来的问题和解决方案]** 有些场景下，通过单参数构造函数隐式的转化成类类型不是我们希望的：例如我们只希望SmallInt类型转化成Number类型，不想通过类型转化序列将可用类型扩大化。为了防止单参数构造函数发生隐式转化，可以使用explicit关键字进行限定：
```c++
class Number{
public:
    explicit Number(const SmallInt &);// 限定SmallInt->Number只能使用显示转化
};
// 编译器不会调用显式的单参数构造函数进行隐式类型转化
SmallInt s_num(12);
extern void func(Number);
func(s_num);// 错误 从SmallInt->Number没有隐式类型转化
func(Number(s_num));// 正确 显示调用Number类型构造函数
func(static_cast<Number>(s_num));// 正确 显式调用Number->SmallInt类型强制转化
```
### 7.3 类型转化函数总结 
<center><img src="/img/in-post/cpp_img/operator_1.pdf" width="100%"></center>

类型转化的相关知识非常重要，尤其是在函数重载解析(cpp primer 3rd chapter9.3)，以及函数模版的重载解析(cpp primer 3rd chapter10.8)中都有重要的应用。<br>
关于类型转化的优先级、类型转化序列产生的二义性处理、类型转化如何划分优先级等问题详见这篇[博客](/cpp/2021/05/22/post-cpp-type-conversion/)中的知识整理。


## 8 取地址操作符重载
下面分别对于取地址操作符的普通版本和const版本进行重载，注意去地址操作符不需要在`op&`参数列表中传入参数，因为`op&`是一个前置单目运算符，他的参数成员为`*this`所以不用显示书写出来。
```c++
// ---------- user.h ----------
class Screen{
    public:
        // overload address-of operators
        Screen* operator&(){return this;}// normal op& overload 
        const Screen* operator&()const {return this;}// const op& overload
    private:
        string _string;
};
// ---------- user.cpp ----------
int main(){
    Screen s;
    Screen *ptr_s=&s;
    return 0;
}
```

## Reference
> 《c++ primer》3rd chapter15 p630 <br>
> https://blog.csdn.net/eastlhu/article/details/80489114 <br>
> https://www.zhihu.com/question/22947192 <br>
> https://www.stroustrup.com/bs_faq2.html#placement-delete <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
