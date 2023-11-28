---
layout: post
title: "cpp - exception"
subtitle: 'cpp中的异常处理相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-15 14:40
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## 1异常处理
异常处理是一种允许两个独立开发的程序组件在程序执行期间遇到异常情况(exception)时，互相通信的机制。本文主要介绍：**1** 如何在程序异常产生的位置产生(raise)或者抛出(throw)异常。**2** 如何通过try-catch代码将程序语句和处理其产生异常的代码相互关联，并且了解catch语句如何处理异常。**3** 介绍异常规范：是一种能够将一组异常和一个函数关联的机制，用于限定程序抛出异常的范围。

## 2异常
异常(exception)是程序可能检测到的，运行时刻不正常的现象。如被0除、数组越界访问、或者自由存储区(空闲存储区)内存耗尽的情况。

### 2.1异常的定义(在类成员函数中的定义)
c++程序中出现异常时，检测到异常的程序段可以通过产生(raise)或者抛出(throw)的方式来通知异常已经发生。下面通过一个自定义的栈程序进行理解：
```c++
#include <vector>
class istack{
    public:
        // constructor
        istack(int capacity):_stack(capacity),_top(0){}
    	// tool function
    	bool pop(int &top_val);// 试图通过返回一个bool来表示操作有无异常
    	bool push(int val);// 试图通过返回一个bool来表示操作有无异常发生
    	bool full();
    	bool empty();
    	void display();
    	int size();
    private:
    	int _top;// 指向栈顶元素在vector中的index
    	vector<int> _stack;// 用于实现istack的容器
};
```
处理上述istack类的对象时，常常发生的错误是：**1** 对一个空的栈执行pop()操作。**2** 对于一个满的栈执行push()操作。我们需要设计相应异常抛出的功能，使得上述异常情况发生时，能够给使用istack类对象的程序、函数发送一个异常通知。下面通过构建一个异常类来实现上述功能：
```c++
// ---------- stackexception.h ----------
class pop_empty{...};// no.1 对空栈执行pop的异常类
class push_full{...};// no.2 对满栈执行push的异常类
```
有了上述异常类的定义，我们可以通过返回一个异常类的对象来告诉istack相应函数调用者：一个xxx类型的异常已经发生了。于是我们修改了上述istack类中pop和push函数返回异常的思路，如下：
```c++
// ---------- istack.h ----------
#include "stackexception.h"
#include <vector>
class istack{
    public:
        ...
    	void pop();// 通过异常类处理相关异常 无需返回bool作为标志
    	void push();
    	...
};
void istack::pop(int &top_val){
    if(empty()){
    	throw pop_empty();// no.1 [异常定义] 抛出pop_empty类型的异常对象
    }
    top_val = _stack[top--];// 将栈顶元素弹出后 栈顶元素index减一
    std::cout<<"istack::pop() - "<<top_val<<std::endl;
}
void istack::push(int val){
    std::cout<<"istack::push() - "<<val<<std::endl;
    if(full()){
    	throw push_full();// no.2 [异常定义] 抛出push_full类型的异常对象
    }
    _stack[++_top]=val;// 栈顶元素index加一 将新元素压栈
}
```
### 2.2try-catch语句进行异常接收&处理
```c++
#include <iostream>
#include "istack.h"
int main(){
    istack stk(32);
    stk.display();
    for(int i=1; i<51; ++i){
    	try{// no.1 异常检测
            if(i%3==0){stk.push(i);}
    	}
    	catch(push_full){...}// 异常处理 no.1
    	if(i%4==0){stk.display();}
    	try{// no.2 异常检测
            if(i%10==0){
            	int tmp;
            	stk.pop(tmp);
            	stk.display();
            }
    	}
    	catch(push_full){...}// 异常处理 no.2 
    }
}
```
上述程序在需要进行异常接收的地方使用try{}语句块进行包含，并且使用catch语句检测try{}语句块中是否发生了指定类型的异常，并进行相应的异常处理。但是，上面的程序有个问题：没有将程序逻辑实现语句和程序异常捕获处理语句相分离，导致程序混乱。采用下面的组织方式较清晰：
```c++
#include <iostream>
#include "istack.h"
in main(){
    istack stk(32);
    stk.display();
    try{// try [异常检测语句]
    	for(int i=0; i<51; ++i){
            if(i%3==0){stk.push(i);}// no.1
            if(i%4==0){stk.display();}
            if(i%10==0){
            	int tmp;
            	stk.pop(tmp);// no.2
            	stk.display();
            }
    	}
    }
    catch(pop_empty){...}// no.1 catch [异常处理语句]
    catch(push_full){...}// no.2 catch [异常处理语句]
    return 0;
}
```
上述简化过的代码执行逻辑如下：<br>
**1** try语句块中没有检测到任何类型的异常，则程序正常执行，try块退出，和try语句块相关的catch异常处理语句被忽略，main函数返回0。<br>
**2** 如果在no.1第一个if语句出检测到异常，则第二第三个if语句不被执行，try语句块退出，直接执行pop_empty异常处理中的代码。<br> 
**3** 如果no.2语句处检测到异常，则display()程序被忽略，跳转到push_full异常处理语句中继续执行。<br>
**4** 假如没有catch语句能够处理该异常，则程序执行权交给**terminate()函数**。

#### 2.2.1try语句块使用规则
**[1] try语句块可以包含任何c++语句(表达式及声明)。** <br>
**[2] try语句块引入一个局部作用域，在try块内部声明的变量不能在外部被使用，即使是catch语句也不行。**

下面整理两种main函数和try-catch语句的组合方式，其中第二种方式较第一种方式较为清晰。
```c++
// ---------- 第一种组织方式 ----------
int main(){
    try{
    	istack stk(32);
    	... // 用于异常检测的代码段
    	return 0;
    }
    catch(err_type1){...}// 异常捕获及异常处理 - stk在catch中不可见
    catch(err_type2){...}
}
```
```c++
// ---------- 第二种组织方式 ----------
int main()try{// 用try将整个main的函数体包含
    istack stk(32);
    ... // 用于异常检测的代码段 
    return 0;
}
// 在main函数体外部进行相应异常检测
catch(err_type1){...}// stk在catch中不可见
catch(err_type2){...}
```
第一种方式不能完全将**异常检测**和**异常捕获&处理**两个部分相分开，而第二种组织方式(称之为'函数try块')可以将两者完全分开，第二种代码组织方式更加清晰。

#### 2.2.2catch语句块使用规则
catch语句由三部分构成：关键字catch、括号中的异常类型声明或者异常对象声明(异常声明)、以及异常处理程序块。
```c++
catch(pop_empty){// 异常声明类型pop_empty
    std::cerr<<"try to pop a value from empty stack!"<<std::endl;
    return errorCode89;
}
catch(push_full){// 异常声明类型push_full
    std::cerr<<"try to push a value into full stack!"<<std::endl;
    return errorCode88;
}
```
上述代码中，异常声明类型分别是pop_empty、push_full。异常声明类型完全匹配，则直接调用相应的异常处理程序块；如果异常声明不匹配，也有可能会调用相应的处理程序块：基类异常声明对应的异常处理程序可以接受派生类异常声明引发的异常。

catch程序块如果不包含return语句，程序的执行将在所有catch语句块后面一条语句进行执行。c++的异常处理机制是不可恢复的(non-resumptive)：一旦异常被处理，程序的执行就不能在异常被抛出的地方继续。

#### 2.2.3catch语句异常对象
前面使用的例子都是使用异常类型声明作为catch语句的参数，并且根据参数调用相应的异常处理语句块。那么什么情况下，使用异常对象作为catch语句的匹配参数呢？<br>
答：当我们想要获取throw表达式中的数值方便异常处理语句使用的时候；当我们需要从异常产生的地方获取一些信息，需要通过throw-try-catch语法进行信息传递的时候；当我们需要将异常信息在异常处理语句块中进行使用的时候；我们就不得不传递一个异常类型的对象(异常信息的载体)。

**[1] 通过throw-try-catch传递异常类型对象通用范式** <br>
通过改变异常类push_full的设计，将不能被压入栈中的数据通过异常类的对象传递给catch语句异常处理模块。
```c++
// ---------- stackexception.h ----------
class push_full{
    public:
    	push_full(int val):_value(val){}// constructor
    	int value(){return _value;}// tool function 
    private:
    	int _value;// 私有数据成员 - 用来保存为能正常压栈的数值
};
// ---------- istack.h ----------
class istack{...};// istack类声明
    istack::push(int val){
    std::cout<<"istack::push() - "<<val<<std::endl;
    if(full()){
    	throw push_full(val);// 显式调用push_full构造函数传值
    }
    _stack[++_top]=val;
}
// ---------- user.cpp ----------
#include <iostream>
#include "istack.h"
int main()try{
    ...// istack类相关操作
}
catch(push_full err_obj){// 能够使用异常类型对象传递的信息提供更加丰富的error提示
    std::cerr<<"trying to push the value"<<err_obj.value()<<std::endl;
}
```

**[2] throw表示式创建异常对象，在全局索引一个变量为其赋予初值。** <br>
异常对象总是在throw异常抛出点被创建，即使throw表达式并不是一个构造函数的形式，或者throw表达式没有表现出要创建一个对象(如同上面err_obj)。
```c++
enum EHstate {noErr, zeroOp, negativeOp, severeError};
enum EHstate state=noErr;
int math_func(int i){
    if(i==0){
    	state=zeroOp;// 这个局部的state没有被作为异常对象
    	throw state;// 创建类型为EHstate的异常对象 用全局state进行值初始化
    }
}
```
上面的例子说明：**(1)** throw表达式总是会创建异常对象。**(2)** throw创建的异常对象总是从全局域进行索引、初始化该对象，局部同名对象被忽略。**(3)** throw创建的对象不是全局对象本身，而是通过在全局域中进行索引同名对象，从而为异常对象赋予初值。

**[3] catch语句中，异常对象通过值传递or引用传递方式进行信息传递**
```c++
enum EHstate {noErr, zeroOp, negativeOp, severeError};
enum EHstate state=noErr;
int math_func(int i){
    if(i==0){
    	state=zeroOp;// 这个局部的state没有作为异常对象
    	throw state;// 创建类型为EHstate的异常对象 用全局state进行值初始化
    }
}
// 值传递
void calculate(int op){
    try{
    	mathFunc(op);
    }
    catch(EHstate err_obj){// throw创建的异常对象通过值传递方式初始化err_obj
    }
}
// 引用传递
void calculate(int op){
    try{
    	mathFunc(op);
    }
    catch(EHstate &err_obj){// throw创建的异常对象通过引用传递方式初始化err_obj
    }
}
```
值传递的方式：err_obj通过异常对象的一个copy的进行初始化。引用传递的方式：err_obj通过直接引用异常对象的方式进行初始化。推荐使用引用的方式为catch语句中的对象进行初始化，这种方式可以有效的避免大型的异常对象拷贝初始化导致的无用的对象创建。

需要注意的是：catch语句使用引用类型的异常声明，使其够修改throw创建异常对象，但是不能够修改throw表达式指定的对象。
```c++
// 引用传递
enum EHstate {noErr, zeroOp, negativeOp, severeError};
enum EHstate state=noErr;// 通过catch修改异常对象 不能影响到全局state
int math_func(int i){
    if(i==0){
    	state=zeroOp;// 未作为异常对象
    	throw state;// 通过catch修改异常对象 不能影响到这个state
    }
}
void calculate(int op){
    try{
    	mathFunc(op);
    }
    catch(EHstate &err_obj){// 通过修改err_obj可以修改throw创建的异常对象
    	err_obj=negativeOp;// 此时throw创建的异常对象被修改
    }
}
```

#### 2.2.4 语句栈展开方式处理异常
一个异常被抛出，寻找对象处理它的catch语句的方式是：**1** 如果抛出异常的位置位于被调函数的try块中，则检查和当前try块相关联的catch语句。如果能找到关联catch语句，则处理该异常；没找到关联的catch语句，则在当前抛出异常的**主调函数**中继续查找。**2** 如果(所有)主调函数退出时带着一个抛出的异常(可能是主调函数引发的，也可能是被调函数传过来的)，并且这个异常位于try块中，检查和其关联的catch语句，能找到则进行处理，否则向这个主调函数的主调函数继续查找。<br>
上述两个过程沿着主调函数被调函数嵌套的方式进行异常处理，直到找到和异常想关联的catch语句。用之前的例子对于异常处理中catch语句的匹配进行理解：
```c++
class push_full{
    public:
    	push_full(int val):_value(val){}
    	int value(){return _value;} 
    private:
    	int _value;
};
// ---------- istack.h ----------
class istack{...};
istack::push(int val){
    std::cout<<"istack::push() - "<<val<<std::endl;
    if(full()){
        throw push_full(val);// 试图向一个满栈压入数据 -> 异常抛出点
    }
    _stack[++_top]=val;
}
// ---------- user.cpp ----------
#include <iostream>
#include "istack.h"
int main()try{
    ...// istack类相关操作 -> main函数调用istack::push函数 -> 产生异常
}
catch(push_full err_obj){// 
    std::cerr<<"trying to push the value"<<err_obj.value()<<std::endl;
}
```
上面代码中，在main函数中的语句执行过程中，在执行istack::push函数时，抛出一个异常，由于该抛出异常的部位没有包含在try语句中，因此在其主调函数main中继续寻找匹配的catch语句。一个相关联的异常处理catch语句被找到，进行异常处理。处理完成后，从处理异常的catch语句集合的下一个语句进行执行程序。

这种因为抛出异常而逐层退出复合语句or函数定义的过程称为**栈展开(stack unwinding)**。栈展开的过程中，复合语句or函数定义中的局部变量生命期结束，被相应的析构函数销毁。如果栈展开完成后还是无法找到对应的catch语句，则执行c++标准库中的函数terminate()，以使程序退出。terminate()缺省参数情况下，调用abort()函数完成程序的非正常退出；terminate()也可以根据[STROUSTRUP97]规则，定义程序退出时的动作。

#### 2.2.5catch语句重新抛出异常
在异常处理过程中，可能存在单个catch语句不能完全处理异常的情况，此时应当重新抛出异常，把异常的处理交给更上层的catch语句进行处理。
```c++
#include <iostream>
#include "istack.h"
int main()try{
    ...// 调用istack类相关成员函数 -> 抛出异常 
}
catch(push_full err_obj){// 
    if(can_handle(err_obj)){...}
    else{
    	throw;// 重新抛出异常 将异常处理交给更上层(主调函数)进行处理
    }
}
```
throw语句重新抛出的异常对象就是catch语句接受的异常对象，通过值传递异常对象的方式，不能在传递过程中修改异常对象：
```c++
#include <iostream>
#include "istack.h"
int main()try{
    ...// 调用istack类相关成员函数 -> 抛出zeroOp的异常类型
}
catch(push_full err_obj){// 
    if(can_handle(err_obj)){...}
    else{
    	err_obj=severeErr;// 试图修改原异常对象
    	throw;// 重新抛出异常 err_obj=zeroOP 重新抛出的异常对象并没有被修改
    }
}
```
上述代码中没有成功修改err_obj的异常类型，重新抛出的异常仍然是原异常类型zeroOP。为了成功修改异常类型，在重新抛出异常，需要使用引用方式进行异常对象的传递：
```c++
#include <iostream>
#include "istack.h"
int main()try{
    ...// 调用istack类相关的成员函数 -> 抛出zeroOP的异常类型
}
catch(push_full &err_obj){
    if(can_handle(err_obj)){...}
    else{
    	err_obj=servereErr;
    	throw;// 重新抛出异常 err_obj=servereErr 异常信息被修改后传递
    }
}
```

#### 2.2.6catch-all捕获所有类型代码
catch-all语句用来捕获并处理一些未知类型的异常情况。catch-all通常和throw重新抛出异常语句结合使用，从而在一个catch语句中执行一些通用的操作，并对某些特定的情况执行定制的操作。<br>
在程序执行一些资源管理释放相关操作时，常常由于异常的发生而导致资源无法正确分配、释放的过程，如下面程序所示：
```c++
void resouce_manage(){
    resource res;
    res.lock();
    ...// 进行一些资源处理操作 -> 可能会抛出一些异常 导致资源不能不释放
    res.release();// 资源释放
}
```
上述程序中，如果资源使用过程产生异常，则资源不能被正确释放。如果通过指针来管理这些资源，由于异常导致程序跳出指针的作用域后，指针变量被释放，但是相应内存资源无法显示进行回收--内存泄漏。因此，我们需要使用catch(...)+throw的方式进行处理：
```c++
void resource_release(){
    resource res;
    res.lock();
    try{
    	...// 执行某些可能会抛出异常的操作 (而且抛出的异常类型不嫩完全确定)
    }
    catch(...){
    	res.release();// 抛出异常的情况下 执行资源释放
    	throw;// 将异常处理交给更上层的主调函数进行处理
    }
    res.release();// try语句块没有抛出异常情况下 执行资源释放
}
```

### 2.3异常规范
异常的规范可以帮助我们判断异常的来源，判断异常的类型，从而更加方便我们进行异常的管理以及代码的优化。

**[通过文本注释的方式记录异常规范]** 在每个可能抛出异常的地方(如之前构造的pop()、push()函数后面添加注释)：
```c++
// ---------- istack.h ----------
class istack{
    ...
    void pop(int &val);// 可能抛出异常pop_empty
    void push(int val);// 可能抛出异常push_full
};
```
上述方式虽然为异常产生的地方做了文档，但是这个文档无法随着istack类型的后续发行而更新；也没有为编译器提供保证不会产生其他类型的异常。

**[throw形式表达式建立异常规范]** 异常规范(exception specification)提供了一种方案：它能够随着函数声明列出可能会抛出的异常；能够保证函数不会抛出其他类型的异常：
```c++
// ---------- istack.h ----------
class istack{
    ...
    void pop(int *val) throw(pop_empty);// 异常声明是函数接口的一部分
    void push(int val) throw(push_full);// 异常声明必须在函数声明中指定
}
```
**[函数的多次声明标识的异常规范不能累积]** 同一函数的异常规范必须制定同一类型的异常类型。同一函数的异常规范类型不同不能累积且导致编译错误。
```c++
extern void func() throw(string);// 异常声明prototype - 异常规范类型为string
extern void func() throw(int);// 错误! 同一函数声明的异常规范类型须相同且不能累积
extern void func();// 错误! 不能省略异常规范
```
**[异常规范的局限性]** 异常规范保证了上述程序中pop、push函数不会抛出pop_empty、push_full之外的异常。需要注意的是：异常规范只是保证了上述程序中不会抛出规范之外的异常，但是他不能保证在执行pop()、push()程序时不会产生异常规范之外的异常。因此，书写异常规范仿佛不能直接解决我们的痛点，throw(xxx)和throw(...)近似等价。简而言之：**异常规范是一种编译期规范，它不能限制在程序运行期抛出异常的类型。** 

**[违反异常规范后的解决方式]** 如果程序产生了一个没有被列在异常规范中的异常怎么办？异常产生、异常的信息传递在程序运行期才能进行，因此即使程序产生了一个异常规范之外的异常，编译器也不会报错。在运行时刻出现的异常规范的违例可能会由如下方式解决：**(1)** 系统调用c++标准库中的函数unexpected()：其中缺省的unexpected()函数将调用terminate()函数终止程序的执行；参考[STROUSTRUP97]标准可以修改缺省inexpected()函数的行为进行特化处理。**(2)**  如果该函数自己处理该异常，并且该异常在逃离该函数之前被处理掉，那么系统将不会调用unexpected()函数处理异常。

**[空的异常类型]** 空的异常类型可以保证函数不会抛出任何类型的异常；而函数声明中没有指定异常规范，则可以抛出任何类型的异常。
```c++
extern void check_func() throw();// 空的异常规范 -> 不抛出任何类型的异常
```
被抛出的异常类型和异常规范中的类型不存在类型转换：
```c++
int convert_exception() throw(string){// 异常规范中声明为string类型异常
    if(...){
    	throw "help!";// 实际显式的抛出一个const char*类型异常 
    }
}
```
上述代码中，异常规范声明了一个string类型的异常，而在函数中实际抛出了一个c风格字符串(const char*)类型的异常，这种对于异常规范进行类型转化是错误的(编译期错误)。可以通过如下方式进行异常规范的声明：
```c++
int convert_exception() throw(string){
    if(...){
    	throw string("help");// 通过改写成利用const char*构造string方式
    }
}
```

**[异常规范在函数指针中的应用]** 函数指针中带有的异常规范写法如下所示：
```c++
void (*func_ptr)(int*, int*) throw(string);
```
上述代码表明，func_ptr是一个函数指针，任何使用它的地方只能抛出string类型的异常。函数指针的异常规范同样不能累积。当带有异常规范的函数指针被初始化或者被赋值的时候，赋值左右两边的异常规范不一定完全相同，但用于初始值or右值的异常规范必须和函数指针一致or更加严格：
```c++
// definitions
void (*func_ptr)(int*, int*) throw(string);
void func1(int*, int*) throw();
void func2(int*, int*) throw(string);
void func3(int*, int*) throw(string, exceptionType);
// usages
void (*func_ptr)(int*, int*) throw(string)=&func1;// right! 更严格!
void (*func_ptr)(int*, int*) throw(string)=&func2;// right! 更严格!
void (*func_ptr)(int*, int*) throw(string)=&func3;// wrong! 编译错误!
```
上述三条函数指针赋值usage程序印证了：只有异常规范更加严格的初始值or右值才能被成功赋值给函数指针。

### 2.4异常设计相关注意事项
这一部分简单介绍异常处理的在具体应用时的注意事项。<br>
**[1]** 异常处理被内置在c++语言中，但是并不是所有的c++程序都应该使用异常处理机制。程序员应当保证只在必要的时候，应用这种机制对于想要解决的异常情况进行相应的捕捉解决。<br>
**[2]** 异常处理应该用于独立开发的程序之间，用于不正常调用、使用情况下的通信。独立开发的程序之间在调用、承接的过程中，往往出现错误而无法完成应有功能。使用异常处理机制，能够进行**运行时通信**，从而更好的定位错误并解决。如：一个库的实现者可能决定用异常向库用户通知程序的异常情况。<br>
**[3]** 类库的作者应当把哪些情况应该抛出异常，哪些情况应该在类库内部解决，哪些情况应当返回错误代码而不是抛出异常，哪些情况应当忽略这个错误并使程序继续执行etc...错误解决方案进行权衡。一个异常处理非常成熟、并且健壮的类库的使用时非常舒服的，虽然对于类库坐着而言，要区分上述几种情况是很棘手的问题。

以istack为例：当我们向一个满栈压入一个数据时，应该匹配上述哪种情况？抛出异常？返回错误码？程序继续执行？终止程序运行？实际上，我们应当'原地'处理这个异常，将push函数写的更加健壮：
```c++
// ---------- istack.h ----------
class istack{...};
istack::push(int val){
    std::cout<<"istack::push() - "<<val<<std::endl;
    if(full()){
    	// throw push_full(val);
    	_stack.resize(2*_stack.size());// 上述异常代码被忽略 改写底层存储器
    }
    _stack[++_top]=val;
}
```
类似的，当我们从一个空栈中pop一个数值时，pop函数是否应该抛出异常？上述几种处理方案如何选择？答：参考c++标准库可知：标准库对此情况不会抛出异常，而是返回这个操作是一个'undefined action'，这是因为不是所有的情况都要通过异常来解决，有的时候返回信息给库使用者并且什么都不做就是最好的处理。

更系统的，大型程序的设计中，往往是通过分成程序层or程序组件的方式进行管理，异常的处理需要在这些模块内部以及外部进行考虑。

## Reference
> 《c++ primer》3rd p454 chapter11 <br>

> **why we need throw? what about exception?** <br>
> https://www.cnblogs.com/whyandinside/p/3677589.html <br>
> http://www.gotw.ca/publications/mill22.htm <br>
> http://stackoverflow.com/questions/88573/should-i-use-an-exception-specifier-in-c/88905#88905 <br>
> http://stackoverflow.com/questions/10787766/when-should-i-really-use-noexcept <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
