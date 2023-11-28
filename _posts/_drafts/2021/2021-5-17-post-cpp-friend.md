---
layout: post
title: "cpp - friend"
subtitle: 'friend友元相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-17 20:05
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## 1友元函数
在某些特定的情况下，只允许一个函数(友元函数)访问类内的成员而不是允许整个程序访问类内成员要好。友元为某个特定的函数访问类内的数据、数据接口提供了方便，但是友元函数破坏了类对于数据、数据接口的封装性(有时为了方便故意使用friend破坏封装性)。继承虽然也被允许访问基类的数据、数据接口，但是这不是对基类封装性的破坏，而是对基类属性本质的一种延伸。

### 1.1什么情况下应当使用友元函数
有三种方式可以定义一个函数，并使用它对类内的数据成员进行操作：**(1)** 将这个函数定义成类的成员函数，成员函数可以直接访问类内任何数据成员。**(2)** 通过定义一个普通函数通过调用类提供的数据成员操作api进行数据访问。**(3)** 通过将一个普通函数在类内声明成类的友元以直接调用类内的任何数据成员。

**[不一定使用友元函数的情况]**下面通过重载operator==操作来比较三种方式，并分析这种情况下是否应该使用友元函数：
```c++
// no.1 普通函数通过string类提供的公有api访问类内的数据
bool operator==(const string &str1, const string &str2){
    if(str1.size() != str2.size()){// 显式的调用string类的公有接口获取数据成员信息
        return false;
    }// strcmp函数通过c_str()接口获取底层c风格字符串
    return strcmp(str1.c_str(), str2.c_str())? false : true;
}
```
```c++
// no.2 类内的成员函数可以直接调用类内数据的 -> 这种调用方式是错误的! 详见no.2 refined
bool string::operator==(const string &str1, const string &str2){
    if(str1._size != str2._size){// 成员函数直接调用类内的数据成员
        return false;
    }// strcmp可以直接访问类内数据成员c风格字符串
    return strcmp(str1._string, str2._string)? false : true;
}
```
```c++
// no.2 refined -> 对于类内定义的operator 编译器只能允许其接受一个参数(强制使用this)
bool string::operator==(const string &str) const{// const *this 
    if(*this._size != str._size){
        return false;
    }
    return strcmp(*this._string, str._string)? false : true;
}
```
```c++
// no.3 将no.1普通函数声明为string类的友元
friend bool operator==(const string &str1, const string &str2);// dec in class
bool operator==(const string &str1, const string &str2){
    if(str1._size != str2._size){
        return false;
    }
    return strcmp(str1._string, str2._string)? false : true;
}
```
实际上，第一种普通函数调用公有api的方式和第三种友元方式效果相近，因为通常情况下，c_str()和size()工具函数会被定义在string类内部，这种情况下，两个功能函数默认是inline的，因此在编译期就已经被展开，使用它们的效率和直接访问私有数据成员类似。这种情况下，将operator==定义成友元是不划算的(未提升效率但是破坏了封装性)。<br>
> **引用c++primer原文对于使用友元函数不必使用的解释:** <br>
> In general, a class implementor should try to minimize the number of namespace functions and operators that have access to the internal representation of a class. <br>
> 一般来说，类的设计者应当尽可能减小使用名字空间中的**function**或者**operator**对于类内部的数据、数据接口进行访问的次数。
简而言之：**能通过类内api完成相应操作，则尽可能设计相关类内api解决。**

上述原则有例外：当类的实现者不想为特定的私有数据成员提供公有的访问api，此时又希望定义在名字空间的function或者operator对私有数据进行访问，则只能通过在类内声明友元函数的方式进行解决。

**[必须使用友元函数的情况]**：为类重载输入输出流操作符。使用类的成员函数进行操作符重载功能上也可以满足要求，但是形式上难以接受：
```c++
// ---------- member func overload ---------- 
istream & operator>>(istream &);// istream declare 
istream & screen::operator>>(istream &in){...}// istream def
ostream & operator<<(ostream &);// ostream declare 
ostream & screen::operator<<(ostream &out){...}// ostream def
// ---------- usage ----------
s.operator>>(cin);  s>>cin;// weird format of cin
s.operator<<(cout);  s<<cout;
```
上面使用类的成员函数重载的方法虽然可行，但是输出输出流重载后使用的方式不符合正常习惯。按照上述重载操作符对于多个数据输出是非常难看的：
```c++
s4>>s3>>s2>>s1>>s>>cin;
s4<<s3<<s2<<s1<<s<<cout;
```
下面是通过将普通函数声明为友元的方式为类screen重载输入输出运算符：
```c++
// ---------- friend func overload ---------- 
friend istream & operator>>(istream &, screen &s);// istream declare 
istream & operator>>(istream &in, screen &s){...}// istream def
friend ostream & operator<<(ostream &, const screen &);// ostream declare 
ostream & operator<<(ostream &out, const screen &s){...}// ostream def
// ---------- usage ----------
operator>>(cin, s);  cin>>s;// equivalent methods 
operator<<(cout, s);  cout>>s;// equivalent methods 
```
使用上述友元重载版本输出多个数据：
```c++
cin>>s>>s1>>s2>>s3>>s4;
cout<<s<<s1<<s2<<s3<<s4;
```

### 1.2友元函数的声明方式
友元函数可能是一个：名字空间中的函数、另外的之前定义的类的成员函数、也可能是一个完整的类(友元类：友元类的所有成员函数都被授权访问该类内的所有成员)。

下面通过代码展示如何在类内声明友元函数：
```c++
class screen{
    // friend func overload
    friend istream & operator>>(istream &, screen &);
    friend ostream & operator<<(ostream &, const screen &);
    // member func overload -> only can take 1 parameter!
    istream &operator>>(istream &);
    ostraem &operator<<(ostream &);
    private:
        int _width; int _height; string _screen;// screen类的其他声明部分
};
```
上述代码中，输入流和输出流函数都接受screen类对象的引用，这么做的原因主要是想要避免不必要的对象拷贝。其中输入流对象定义成一个引用：从输入流捕捉数据后可能会对其进行修改，而输出流对象定义成常量引用：输出的对象不允许修改。友元函数可以访问screen内的所有成员：
```c++
ostream & operator<<(ostream &out, const screen &s){
    out<<"<"<<s._height<<","<<s._width<<">"<<endl; 
    out<<s._screen;
    return out;
}
```
如果希望名字空间函数同时具有访问多个类别的权限，那么应当把这个函数在多个类中声明称友元函数：
```c++
class window;// declare for screen
class screen{
    friend bool is_equal(screen &, window &);// friend 
};
class screen{
    friend bool is_equal(screen &, window &);// friend
};
```
如果希望这个函数是一个类的成员函数，同时又是另一个类的友元函数：
```c++
class window;
class screen{
    screen & copy(window &);// member func copy
};
class window{
    friend screen & screen::copy(window &);// access by ::
};
```
如果希望两个类的成员函数互为友元，应该如何声明？使用上一个办法是通常很难实现：只有编译器'看到'一个类的定义时，它的成员函数才能被声明为其他类的友元。因此，这种情况使用友元类进行处理：
```c++
class window;
class screen{
    friend class window;
};
class window{
    friend class screen;
};
```
有了上述声明，现在window和screen两类的成员函数可以互相访问数据成员、api。

## 2友元类 - todo

## Reference
> 《c++ primer》3rd chapter15 p614

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
