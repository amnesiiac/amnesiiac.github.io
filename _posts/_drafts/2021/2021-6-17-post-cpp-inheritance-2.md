---
layout: post
title: "cpp - inheritance-2"
subtitle: '关于继承的用法相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-06-17 14:51
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## 继承的使用
**借助继承机制，通过指向基类类型的指针或引用指向派生类的对象，可以使基类的指针拥有操作多个派生类对象的能力，称之为多态**。<br>
需要注意的是：c++中只允许指向基类的指针or引用**直接指向**派生类的对象，但是不允许指向派生类的指针或引用**直接指向**基类类型的对象。可以使用强制类型转化(static_cast和dynamic_cast)将派生类的指针转化成基类指针类型。

本文内容主要分成三个部分：**1** RTTI：让程序能够获取基类指针or引用在运行时指向的对象的派生类型。**2** 异常机制在类的继承树中如何使用。**3** 类型的继承机制对于函数重载类型解析中类型转化部分的影响。下面依次对于这三个部分进行介绍：

## RTTI - runtime type identification
**[1] 什么是运行时刻类型识别(RTTI)** <br>
运行时刻类型识别：用基类指针or引用来操纵对象的程序能够在运行期间获取到这些指针or引用所指向对象的实际派生类型。这种机制主要依赖两个操作符完成：dynamic_cast和typeid。

**[2] RTTI的两个操作符dynamic_cast和typeid**<br>
dynamic_cast 允许运行时刻的类型转换(基类指针类型转换成派生类的指针、基类的左值转化成派生类的引用)，并且它保证了在类层次中进行类型转换是安全的。typeid指出指针or引用指向的对象的实际派生类型。

需要注意的是：RTTI操作符(dynamic_cast和typeid)对于带有一个或多个虚函数的类类型而言是运行时刻进行类型检查；对于其他类型而言，只进行编译时刻类型检查。

### dynamic_cast
dynamic_cast操作符可以将一个类类型对象的指针转化成同一类层次中其他类的指针；也可以将一个类类型对象的左值转换成同一类层次结构中其他类的引用。如果这两种转化不能成功，则针对指针类型的转化结果是nullptr(0)，针对引用类型的转化结果是抛出一个异常(exception)。

**[#1 使用dynamic_cast将基类指针转化成派生类的指针]**
```c++
class employee{
    public:
        virtual int salary();
};
class manager: public employee{
    public:
        int salary();
};
class programmer: public employee{
    public:
        int salary();
};
void company::payroll(employee *pe){// pe是一个基类类型的指针
    // 基类的指针可以转化成派生类的指针 从而调用manager或programmer类的成员
    // pe->salary();
}
```
现在，我们进一步修改上述类继承结构模型，向基类employee类增加一个bonus()成员：
```c++
class employee{
    public:
        virtual int salary();
        virtual int bonus();
};
class manager: public employee{};
class programmer: public employee{
    public:
        int salary();
        int bonus();
};
void company::payroll(employee *pe){
    // 按照参数传入的类型(manager或programmer) 调用不同的bonus函数
    // 传入manager则调用employee::bonus  传入employee则调用programmer::bonus
    // pe->bonus();
}
```
需要注意的是：类继承层次模型添加了一个虚函数之后，我们需要重新编译类层次结构中所有类成员函数，但这种方式需要能够访问类层次结构的源代码，which is not always realistic。当我们不能直接访问类继承层次模型源代码时(准确的说是我们不能直接访问基类的源代码，只能修改派生类的源代码)，则需要通过dynamic_cast为类继承结构增加功能。
```c++
class employee{// read-only
    public:
        virtual int salary();
};
class manager: public employee{// accessible
    public:
        int salary();
};
class programmer: public employee{// accessible
    public:
        int salary();
        int bonus();// add bonus method
};
void company::payroll(employee *pe){
    programmer *pm = dynamic_cast<programmer*>(pe);// 将基类指针转化成派生类指针
    if(pm){// 如果pm!=nullptr -> 转换成功 pm指向programmer对象
        pm->programmer::bonus();
    }
    else{// 如果pm==nullptr -> 转化失败
        // use employee成员函数
    }
}
```
上述代码中company::payroll()中的dynamic_cast能否转化成功取决于pe是否接受了一个programmer类型的对象地址：如果pe接受了一个programmer类型的对象地址，则转化成功，否则转化失败pm=nullptr。**简而言之，pe能否转化成功取决于被转化的操作数的底层类型和转化目标底层类型是否匹配。**关于dynamic_cast的转化机制可以参考[博客](/cpp/2021/05/01/post-cpp-type-cast/#dynamic_cast)中的介绍。

**[关于dynamic_cast作用]** 上述代码中，dynamic_cast被用来执行从基类指针到派生类指针的安全转化，常常被称为安全的downcasting。当我们需要使用派生类的特性，而基类又不具备该特性的时候，有两种机制可以应对这种场景：虚函数机制、dynamic_cast。在某些情况下，不能为基类添加虚函数并重新编译基类源代码，只能为派生类定义函数方法，然后借助dynamic_cast进行downcast方式予以解决。使用dynamic_cast相比虚函数机制更容易出错，使用时需要谨慎(一定要检查是否转化成功)。

**[#2 使用dynamic_cast将基类的左值转化成派生类的引用]**
```c++
class employee{// read-only
    public:
        virtual int salary();
};
class manager: public employee{// accessible
    public:
        int salary();
};
class programmer: public employee{// accessible
    public:
        int salary();
        int bonus();// add bonus method
};
void company::payroll(employee &re){
    try{
        programmer &rm = dynamic_cast<programmer&>(re);// 可能抛出异常
        // 使用派生类programmer类的成员函数 
    }
    catch(std::bad_cast){
        // 使用基类的成员函数
    }
}
```
上述代码中，使用try-catch结构捕获dynamic_cast进行基类引用转化成派生类左值时可能抛出的bad_cast异常。注意，由于不存在引用为空，因此不能使用if-else对于转化失败的情况进行处理。另外，bad_cast类型异常包含在typeinfo中，需要包含该头文件才能使用。最后，使用引用方式的dynamic_cast由于需要借助异常进行结果检查，异常机制会增加程序运行的开销，使用哪种方式需要程序员进行权衡。

### typeid
**[#1 typeid使用的三种场景]** <br>
- typeid接受非类类型参数 -> 输出操作数类型
```c++
int obj;
std::cout<<typeid(obj).name()<<std::cout;// 输出 "int"
std::cout<<typeid(0.18).name()<<std::cout;// 输出 "double"
```
- typeid接受类类型参数，但是该类类型不带有虚函数 -> 输出操作数类型而不是底层对象类型
```c++
class Base{/* 不含虚函数 */};
class Derived: public Base{/* 不含虚函数 */};
Derived obj;
Base *ptr = &obj;
Base &ref = obj;
// 输出 操作数类型"Base" 而不是底层对象类型"Derived"
std::cout<<typeid(*ptr).name()<<std::endl;
// 输出 操作数类型"Base" 而不是底层对象类型"Dervied"
std::cout<<typeid(ref).name()<<std::endl;
```
- typeid接受类类型参数，该类类型带有虚函数 -> 输出底层对象类型而不是操作数类型
```c++
class Base{/* 含有虚函数 */};
class Derived: public Base{/* 不含虚函数 */};
Derived obj;
Base *ptr = &obj;
Base &ref = obj;
// 输出 底层对象类型"Dervied" 而不是操作数类型"Base"
std::cout<<typeid(*ptr).name()<<std::endl;
// 输出 底层对象类型"Derived" 而不是操作数类型"Base"
std::cout<<typeid(ref).name()<<std::endl;
```
- 关于typeid上述情况的补充 -> typeid实际返回的是一个type_info类型的对象
```c++
class Base{/* 含有虚函数 */};
class Derived: public Base{/* 不含虚函数 */};
Derived obj;
Base *ptr = &obj;
Derived &ref = *ptr;
// 输出 "true" *ptr是一个类类型的对象 -> 底层
std::cout<<typeid(*ptr)==typeid(Derived)<<std::endl;
// 输出 "true" ptr是一个指针类型 -> 操作数
std::cout<<typeid(ptr)==typeid(Base*)<<std::endl;
// 输出 "true" ref是一个类类型对象的引用 -> 底层
std::cout<<typeid(ref)==typeid(Derived)<<std::endl;
// 输出 "true" &ref是类类型对象的地址 -> 操作数
std::cout<<typeid(&ref)==typeid(Base*)<<std::endl;
```

**[#2 typeid的用法]** <br>
> cpp primer 3rd chapter 19.1.2 p850 <br>
> typeid可以用在高级系统程序设计开发中，如:**(1)** 构造调试器的设计过程(building debuggers)或者**(2)** 用于保证从数据库中获取的永久性对象的正确性。在这样的系统中，当一个程序通过一个基类的指针或引用来操纵程序中的对象时，程序本身需要在**(1)** debug session中或者**(2)** 从数据库中retrieve数据时，正确地获取被操纵对象的真实类型以正确地列出对象的各项属性。

**[#3 type_info类]** <br>
type_info类的定义 - 在程序中创建type_info对象的唯一方式是typeid操作符：
```c++
class type_info{
    private:// 将copy ctor和copy assign op定义为私有成员 禁止在类域外定义对象
        type_info(const type_info&);
        type_info& operator=(const type_info&);
    public:
        virtual ~type_info();
        int operator==(const type_info&) const;
        int operator!=(const type_info&) const;
        const char* name() const;
};
```
type_info类的具体实现方式是和编译器相关的，返回类型名的方法(name)是唯一保证被所有编译器支持提供的信息。如果需要使用特定编译器提供的type_info方法，需要查阅编译器手册来找到RTTI的特殊支持。


## 异常和继承
异常处理是针对运行时刻的程序异常(runtime exception)而提供的语言层次的标准设施。c++为异常的处理提供了一套完整的语法风格，同时兼容程序员对异常处理细节设施进行调整。关于异常机制的基本知识在[博客](/cpp/2021/05/15/post-cpp-exception/)中已经有较详细的介绍，这部分内容主要介绍较为复杂的类类型层次结构中的异常的实施和注意事项。

### 使用类层次结构定义异常
在实际的c++程序中，异常类的代码通常使用组(group)或层次结构(hierarchies)进行组织，如下所示：
```c++
class ExceptionBase{...};
class PopOnEmpty: public ExceptionBase{...};
class PushOnFull: public ExceptionBase{...};
```
当然，我们可以使用更加精细的方式组织上述类层次结构的异常类：
```c++
class ExceptionBase{...};
    class StackException: public ExceptionBase{...};
        class PopOnEmpty: public StackException{...};
        class PushOnFull: public StackException{...};
    class MathException: public ExceptionBase{...};
        class ZeroOp: public MathException{...};
        class DivideByZero: public MathException{...};
```
这种精细化的类层次结构异常处理能够应对更加复杂的程序中的异常场景。

### 抛出类类型的异常
**[1] throw抛出、传递异常的基本流程 a->b->c**
```c++
void iStack::push(int value){
    if(full()){
        // (a) throw表达式创建PushOnFull类的临时对象
        // (b) 创建一个PushOnFull的异常对象 并将no.1中临时对象拷贝给异常对象
        // (c) 在进行异常处理的栈展开之前 销毁no.1创建的临时对象
        throw(PushOnFull(value));
    }
}
```
**为什么要创建异常对象?异常对象有什么用?** <br>
异常对象是用来将异常信息向上层调用链进行传递的媒介。<br>
临时对象在throw表达式结束时销毁，但是异常信息需要传递给函数调用链上层的函数用于异常处理，因此，需要将异常信息保存在异常对象相关storage中，直至异常处理完毕或程序执行terminate()。另外，对于throw中临时对象的销毁is not required or always possible。

**[2] throw抛出异常的具体类型**
```c++
void iStack::push(int value){
    if(full()){
        PushOnFull except(value);// 派生类异常临时对象 -> need ctor(int)
        StackException *ptr = &except;// 基类指针downcasting实现多态
        // 异常对象类型=StackException 
        // 异常对象中包含的异常信息需要从except中进行拷贝
        // "多态的"调用PushOnFull::copy_ctor -> 将except中异常信息拷贝给基类异常对象
        throw *ptr;
    }
}// need PushOnFull dtor 
```
上述代码中，throw表达式指向对象的类型是StackException，而不是PushOnFull。因此throw表达式创建的异常对象是StackException类型，即所捕获的异常不能被PushOnFull类型catch子句处理。<br>
另外，如果存在如下几种情形，则throw表达式为错误的：
- PushOnFull类型没有能够接受int类型的构造函数：except对象不能被创建。
- PushOnFull类型的拷贝函数和析构函数不能访问：except对象不能被创建。
- PushOnFull是一个抽象基类：抽象基类无法创建对象，except对象不能被创建。

**关于上述结论，有几个问题需要讨论：**<br>
**Question** 为什么throw表达式指向对象类型是基类类型，而不是派生类类型，是因为构建异常类层次结构时使用的是非虚函数的方式么？<br>
**Answer**：是的，一般而言，throw表达式抛出的异常类类型层次中，不使用虚函数在不同层次的异常处理中实现多态。但特殊情况，如catch(基类异常类型)处理时，希望能够利用派生类异常类型的特性，才通过虚函数机制构建异常类层次结构。(详见cpp primer 3rd 19.2.4)

**Question** 假设异常对象类型为基类类型，那么throw表达式不是应该调用基类类型的相关copy ctor和dtor进行构造析构么，为什么要求派生类类型的copy ctor和dtor能够访问？<br>
**Answer** 派生类异常类型PushOnfull的copy ctor和dtor可以访问是因为：创建完except异常临时对象后，需要将except临时对象拷贝给基类异常对象，因此需要"多态的"调用PushOnFull::copy_ctor，销毁except临时对象时需要调用~PushOnFull()。

### 处理类类型的异常 
**[1] 用于类层次结构的异常捕获的catch子句的组织方式** <br>
当异常被组织成类层次结构时，类类型的异常可能会被被该类型的公有基类(:public Base)的catch子句捕获到，如下：
```c++
int main(){
    try{/*...*/}
    catch(Exception){// 既可以捕获PushOnFull类型异常 也可以捕获PopOnEmpty类型异常
        // 处理PushOnFull和PopOnEmpty类型异常
    }
    catch(PushOnFull){// 只能捕获PushOnFull及其子类型异常
        // 处理pushOnFull类型异常
    }
}
```
上述代码中异常捕获catch子句的组织方式有问题：try-catch语句中，catch子句按照顺序执行，只要有一个catch执行,就不会进一步检查同级其他catch子句。由于catch(Exception)能捕获其子类型异常，则catch(PushOnFull)子句永远不会执行。修正后的代码如下：
```c++
int main(){
    try{/*...*/}
    catch(PushOnFull){// 在异常处理过程中 最特化的catch子句必须最先出现
        // 处理pushOnFull类型异常
    }
    catch(Exception){
        // 处理PushOnFull和PopOnEmpty类型异常
    }
}
```
catch子句的设计应当按照从派生类到基类的顺序排列。进而，当异常处理程序被构建成类层次结构时，类库的用户就可以主动选择用于异常捕获处理的粒度和层次。
```c++
// no.1 下面catch括号中的异常声明 和用实参值拷贝初始化一个按值传递的函数参数相同
catch(PushOnFull obj){// 在细粒度层次处理异常
    std::cerr<<"push the value "<<obj.value()<<" on a full stack"<<std::endl;
}
// no.2 下面catch括号中的异常声明 和用实参的引用初始化一个函数参数相同
catch(PushOnFull &obj){// 在细粒度层次处理异常
    std::cerr<<"push the value "<<obj.value()<<" on a full stack"<<std::endl;
}
catch(Exception){// 在基类excetion层次处理异常
    Exception::print("an exception was encountered");// 调用基类成员函数print
}
```
**[2] 类层次结构异常的重新抛出throw();** <br>
```c++
void calculate(int var){
    try{
        mathfunc(var);// throw DivideByZero exception obj
    }
    // 上述异常被基类MathException类型catch子句捕获
    // obj=copy of the MathException subobject
    // the MathException subobject is contructed by DivideByZero exception obj
    catch(MathException obj){
        // no.1 部分处理该异常
        // no.2 然后重新抛出该异常 重新抛出的异常是什么类型？
        // no.2 是捕获catch子句中的异常类型? 还是try块抛出的异常类型?
        // no.2 无论什么情况 总是重新抛出try块中的异常类型: DivideByZero 
        throw;// rethrow exception
    }
}
```
**[3] 类层次结构异常类型的析构函数调用时机** 定义PushOnFull类型的异常类型如下：
```c++
class PushOnFull{
    public:
        PushOnFull(int i):_value(i){}
        int value(){return _value;}
        ~PushOnFull();// 异常类型的dtor
    private:
        int _value;
};
```
为了确定调用异常类型的析构函数的时机，需要考虑catch子句：
```c++
try{
    // 抛出异常 -> 调用类层次异常类型的ctor构建异常对象 -> 
    throw(PushOnFull excp);
    // no.2 直到处理该异常的最后一个catch子句退出时才调用异常类型的dtor将其销毁
    // 最后一个catch子句: 需要考虑被某一层次的catch捕获后重新抛出到上一级调用的情况
}
catch(PushOnFull obj){// no.1 obj=catch子句中的局部对象 -> catch子句结束时被自动销毁
    std::cerr<<"push the value "<<obj.value()<<" on a full stack"<<std::endl;
}
```

### 异常对象类层次和虚函数机制 
如果try块中抛出的异常是异常类层次中派生类型的，当这个所构建的异常对象被基类异常类型对应的catch子句捕获，则这个catch子句一般不能使用派生类独有的特性：
```c++
catch(Exception &obj){
    // wrong! Exception类没有value()函数
    std::cerr<<"push the value "<<obj.value()<<" on a full stack"<<std::endl;
}
```
针对上述代码中的问题，一种解决方案是：重新设计异常类层次结构，将value()函数在基类中定义成虚函数，并通过基类的虚函数来调用派生类中相应特化的函数：
```c++
class Exception{
    public:
        virtual void print(){
            std::cerr<<"an exception occured"<<std::endl;
        }
};
class StackException: public Exception{};
class PushOnFull: public StackException{
    public:
        virtual void print(){
            std::cerr<<"push the value "<<_value<<" on full stack"<<std::endl;
        }
};
```
有了上述的定义了print虚函数的异常类层次结构定义，print函数可以用在catch(Exception)对象处理的程序块中：
```c++
int main(){
    try(){
        iStack::push();// throw a PushOnFull exception obj
    }
    // obj = <copy> of Exception subobject, constructed by PushOnFull obj 
    catch(Exception obj){
        obj.print();// ok! 调用异常层次基类的虚函数print "an exception occured"
    }
}
```
为什么调用的是基类Exception的print函数而不是派生类PushOnFull的print函数呢？这是因为catch子句中异常声明行为和函数参数声明非常相似，上述catch子句中的obj是异常对象的一个拷贝，属于Exception类型，因此调用接类print函数。为了使catch子句调用派生类对象的虚函数，异常声明须使用引用、指针类型：
```c++
int main(){
    try(){
        iStack::push();// 抛出PushOnFull exception obj
    }
    catch(Exception &obj){// 将catch子句中异常声明为引用
        // [Question] 为什么这种声明形式可以实现多态调用？
        // [Answer] Exception &obj = PushOnFull对象 -> 根据RTTI typeid:
        //          typeid(obj) == typeid(PushOnFull) 
        // [Attention] 这种多态调用需要有Exception虚函数作为支持
        obj.print();// 调用virtual PushOnFull::print()
    }
}
```

### 栈展开和析构函数调用
**[异常catch子句匹配过程中的栈展开]** 当异常被抛出，为了寻找能够匹配该异常的catch子句，需要从抛出异常的函数内部开始，向上查找函数调用链，直到找到和该异常相匹配的catch子句，或找不到相互匹配的异常而调用terminate()结束程序。在向上查找函数调用链的过程即是栈展开的过程(stack unwinding)。<br>
**[栈展开中资源释放可能出现的问题]** 栈展开的过程中，随着函数调用链中的每个函数在catch子句查找过程中退出，每个函数的执行过程被中断。如果这个函数已经申请了资源(打开一个文件or在空闲存储区申请了空间)，那么这部分资源在程序运行期间永远不会被释放。<br>
**[资源申请即是初始化、资源释放即是析构(RAII)]**是上述问题的解决方案。
```c++
class Ptr{
    public:
        Ptr(){ptr = new int[chunk];}// 资源申请即是初始化
        ~Ptr(){delete[] _ptr;}// 资源释放即是析构
    private:
        int *_ptr;
};
void manip(int para){
    Ptr localptr;// no.3 mathfunc之前创建的局部对象localptr
    ...
    mathfunc(para);// no.1 抛出DivideByZero类型异常 跳转至no.2处退出manip函数
    ...// 未被执行
}// no.2 销毁所有在mathfunc函数之前创建的局部类对象 然后执行manip函数中断退出 
```
上述代码中，mathfunc中抛出异常，函数manip被检查到，但是由于mathfunc没有包含在try块中，所以不会在manip中查找该异常catch子句，而是继续向上遍历函数调用链、查找调用manip的函数。在退出manip函数之前，会调用所有抛出异常之前创建的局部对象的析构函数。如果相应析构函数没有释放之前申请的资源，则该资源在程序运行期间永远不会被释放。<br>

### 异常规范(exception specification)
通过在函数声明中指定异常规范，可以指定函数在编译期中直接、间接抛出的异常集合；并且异常规范可以保证编译期不抛出任何异常。

**[1] 异常规范使用的注意事项** <br>
**(a)** throw(...)异常规范需要写在const/volatile之后。<br>
**(b)** 同一函数所有声明、定义中的异常规范需要保持一致。 
```c++
class bad_alloc: public exception{
    public:
        // no.2 同一函数所有声明、定义中的异常规范须保持一致
        bad_alloc() throw();// no.2 函数声明中的异常规范
        bad_alloc(const bad_alloc&) throw();
        bad_alloc& operator=(const bad_alloc&) throw();
        virtual ~bad_alloc() throw();
        // no.1 throw写在const/volatile之后
        virtual const char* what() const throw();
};
bad_alloc::bad_alloc() throw{// no.2 函数定义中的异常规范 须和声明中相统一
    ...
}
```
**(c)** 基类中的虚函数的异常规范，和派生类改写的成员函数异常规范不同；但是派生类虚函数的异常规范必和基类虚函数异常规范一样or更严格。
```c++
class Base{// 基类虚函数异常规范
    public:
        virtual double f1(double) throw();
        virtual int f2() throw(int);
        virtual string f3() throw(int, string);
};
class Derived: public Base{// 派生类虚函数异常规范
    public:
        double f1(double) throw(string);// wrong! 异常规范没有一致or更严格
        int f2(int) throw(int);// ok! 异常规范相同
        string f3() throw(int);// ok! 异常规范更加严格
};
```
保证派生类成员函数异常规范至少和基类一致的原因在于：派生类的虚函数通过基类的指针进行调用时，该调用保证不会违背基类的异常规范：
```c++
// 多态机制下 派生类异常规范需要比基类更加严格 才能保证基类api函数正常运行
// compute函数是基类的api函数 异常规范参考基类标准进行设计
void complete(Base *ptr) throw(){// throw保证不抛出异常
    try{
        // 根据ptr指向的对象类型不同: ptr->Base::f3() or ptr->Derived::f3()
        // Base::f3() 可能抛出int string 
        // Derived::f3() 可能抛出int类型异常(不会违背异常规范)
        ptr->f3();
    }
    // 处理来自Base::f3()的异常
    catch(const string&){}
    catch(const int){}
}
```
**(d)** 抛出的异常类型可以进行类型转换(特例)
一般情况下，**被抛出的异常类型**和**指定的异常类型之间**不允许类型转换。但是上述规则有一个特例：当异常规范指定了一个类类型or类类型的指针时，允许函数抛出**公有派生的类类型对象**or**公有派生类类型指针**。
```c++
class StackException: public Exception{};
class PopOnEmpty: public StackException{};
class PushOnFull: public StackException{};
void stackmanip() throw(StackException){// 异常规范指定了基类类型异常
    ...// 可以抛出公有派生类型异常PopOnEmpty PushOnFull
}
```

### 构造函数和函数try块
**[1] 函数try块** 将函数体整个包含在try块中，可以在函数体之后进行异常处理。
```c++
int main() try{
    ...// 可能抛出异常
}
catch(PushOnFull){...}// 在main函数体外边处理异常
catch(PopOnEmpty){...}
```
**[2] 函数try块对于捕获构造函数中的异常是必须的**
```c++
inline Account::Account(const char* name, double opening_bal):
    _balance(openint_bal-Service_Charge()){// 调用函数初始化(该函数可能会抛出异常)
    _name = new char[strlen(name)+1];
    strcpy(_name, name);
    _acct_no = get_unique_acct_no();
}
```
上述Account类构造函数在使用初始化列表初始化\_balance成员时，需要调用*Service_Charge()*，而该函数可能在运行时抛出一个异常。为了处理这个函数抛出的异常，将try块包含在构造函数体中是不可行的。
```c++
inline Account::Account(const char* name, double opening_bal):
    _balance(openint_bal-Service_Charge()){// 调用函数初始化(该函数可能会抛出异常)
    try{// 在构造函数体中建立try-catch
        _name = new char[strlen(name)+1];
        strcpy(_name, name);
        _acct_no = get_unique_acct_no();
    }
    catch(...){
        ... // 针对特定异常及其子类进行处理 -> 无法处理_balance初始化异常
    }
}
```
由于上述代码try块不包含构造函数的初始化列表，因此不能对于初始化函数列表中的异常进行处理。使用函数try块可以解决上述问题：
```c++
inline Account::Account(const char* name, double opening_bal) try:
    _balance(openint_bal-Service_Charge()){// 调用函数初始化(该函数可能会抛出异常)
    _name = new char[strlen(name)+1];
    strcpy(_name, name);
    _acct_no = get_unique_acct_no();
}
catch(...){
    ... // 可以处理构造函数初始化列表中、构造函数体中的异常
}
```

### c++标准库中的异常类层次结构
使用自定义的异常类层次结构可以自定义异常捕获的层次、时机。<br>
**[1] std Exception类异常** <br>
c++标准库也提供了一种异常类层次(exception)用于报告标准库执行过程中发生的程序运行不正常情况。标准库异常支持进一步派生，以描述自定义的异常情况。
```c++
namespace std{// 异常名字都放在名字空间std中 防治名字污染
    class exception{// exception异常基类
        public:
            exception() throw();// 默认构造
            exception(const exception&) throw();// 拷贝构造
            exception& operator=(const exception&) throw();// op=
            virtual ~exception() throw();// 析构函数
            // what虚函数 通过派生类进行重载可以用于抛出异常时显示异常层次
            virtual const char* what() const throw();
    };
}
```
**[2] Exception派生的异常类型 --- logic error & runtime error** <br>
**逻辑错误(logic_error)** 是由于程序内部逻辑导致的错误。逻辑错误可以避免，并且在程序运行之前就可以检测到。标准库中逻辑错误相关的异常类层次结构如下：
```c++
namespace std{
    class logic_error: public exception{// 1 逻辑错误 -> exception
        public:// forbid implicit conversion
            explicit logic_error(const string &what_arg);
    };
    class invalid_argument: public logic_error{// 1.1 无效实参 -> logic_error
        public:
            explicit invalid_argument(const string &what_arg);
    };
    class out_of_range: public class logic_error{// 1.2 范围异常 -> logic_error
        public:
            explicit out_of_range(const string &what_arg);
    };
    class length_error: public class logic_error{// 1.3 长度错误 -> logic_error
        public:
            explicit length_error(const string &what_arg);
    };
    class domain_error: public class logic_error{// 1.4 域错误 -> logic_error
        public:
            explicit domain_error(const string &what_arg);
    };
}
```
**Exception派生的异常类型 --- 运行时刻异常(runtime_error)** <br> 运行时刻错误是指程序域之外产生的错误。运行时刻错误只有在程序执行时才能被检测。标准库中定义的运行时刻错误类层次结构如下：
```c++
namespace std{
    class runtime_error: public exception{// 2 运行时刻异常 -> exception
        public:
            explicit runtime_error(const string &what_arg);
    };
    class range_error: public runtime_error{// 2.1 计算范围错误 -> runtime 
        public:
            explicit range_error(const string &what_arg);
    };
    class overflow_error: public runtime_error{// 2.2 计算上溢错误 -> runtime
        public:
            explicit overflow_error(const string &what_arg);
    };
    class underflow_error: public runtime_error{// 2.3 计算下溢错误 -> runtime
        public:
            explicit underflow_error(const string &what_arg);
    };
}
```
**Exception派生的异常类型 --- bad_alloc & bad_cast** <br>
exception同时也是bad_alloc和bad_cast的基类。当new operator不能正确的分配所需内存时，它会抛出一个bad_alloc异常。当使用dynamic_cast进行类型转化失败时，程序会抛出bad_cast异常。

**使用Exception子类out_of_range异常的例子** <br>
```c++
#include<stdexcept> // 使用预定义的异常类 需包含的头文件
#include<string>
template<typename eletype> class Array{// 定义Array类 重载op[] 抛出异常
    public:
        eletype& operator[](int ix) const{
            if(ix<0 || ix>=size){
                string obj="out of range error in Array<eletype>::operator[]";
                throw out_of_range(obj);// 抛出out_of_range类型异常
            }
            return _ia[ix];
        }
        ...
    private:
        int _size;
        eletype *_ia;
};
// 使用上述定义的异常类型
int main(){
    try{
        ...// Array数组访问索引越界 抛出out_of_range类型异常
    }
    catch(const out_of_range &excp){
    // out_of_range error in Array<elemType>::operator[]()
        std::cerr<<excp.what()<<std::endl;
        return -1;
    }
}
```

## 类继承层次关系中的重载解析
在之前的[博客](/cpp/2021/05/11/post-cpp-overload/)中，介绍了关于函数重载解析的相关内容。本文主要围绕类继承层次结构中的重载解析相关内容进行介绍。类层次结构会影响到成员函数重载解析机制中的各个方面：<br>
- 影响选择候选函数选择范围：基类的成员函数、和基类被定义的名字空间中的函数都将被考虑。
- 影响可行函数中实参类型转换范围：实参类型转换将会额外考虑用户自定义的类型转化。
- 影响选择最佳匹配函数的匹配等级：类的继承机制会影响到实参类型转换的优先级列表。

### 确定候选函数
**[1] func(args)** <br> 普通函数(func)实参(args)类型为类类型or类类型的引用or类类型的指针，并且相应类类型都是在同一个名字空间被定义的，则该名字空间中声明的、和被调函数同名的函数都是候选函数，即使这些函数在调用点上不可见(详见cpp primer 3rd chapter15.10)。<br>
进一步地，考虑继承机制：如果args为类类型or类类型的引用or类类型的指针，并且该类有基类，则定义基类的名字空间中的声明的、和被调函数同名的函数也是候选函数，即使这些函数在调用点上不可见。
```c++
namespace NoName{
    class zooAnimal{...};// Bear类的基类zooAmianl的定义
    void display(const zooAnimal&);// 也是候选函数 即使在调用点不可见
}
class Bear: public NoName::zooAnimal{...};// Bear类的定义
int main(){
    Bear dodo;
    // no.1 候选函数不仅仅包括调用点可见的同名函数 
    // no.2 还包括声明Bear类、声明zooAnimal类的名字空间中的函数
    display(dodo);// 调用点
    return 0;
}
```
另外，考虑友元函数机制：如果args为类类型or类类型的引用or类类型的指针，并且该类定义了和调用函数同名的友元函数，则这些友元函数也是候选函数，即使这些函数在调用点上不可见。<br>
进一步地，如果考虑继承机制：如果args为类类型or类类型的引用or类类型的指针，并且该类有基类，则该类及其基类中声明的同名友元函数也是候选函数，即使这些函数在调用点上不可见。
```c++
namespace NoName{
    class zooAnimal{// Bear的基类中的友元函数是候选函数
        friend void display(const zooAnimal&);// 将display声明为友元函数
    };
}
class Bear: public NoName::zooAnimal{...};// Bear类的同名友元函数是候选函数 
int main(){
    Bear dodo;
    display(dodo);// 调用点
    return 0;
}
```
综上，普通函数调用实参是类类型的对象、类类型的指针、类类型的引用，则候选函数是如下三种集合的并集：
- 在调用点可见的函数
- 在定义该类型的名字空间中函数or定义该类的基类的名字空间中的函数
- 该类类型或其基类类型的声明的友元函数



**[2] obj.member_func(args) & ptr->member_func(args)** <br>
对于使用&nbsp;**.**&nbsp;以及&nbsp;**->**&nbsp;操作符访问的函数，实参类类型的继承关系对候选函数查找范围的影响和普通函数[1]不同。其原因是：派生类的成员函数声明并不会重载基类的声明的同名函数，相反派生类的成员函数隐藏了基类中同名成员函数的声明(即使两者函数列表不同)：
```c++
class zooAnimal{
    public:
        Time feeding_time(string);// 基类同名成员函数
};
class Bear: public zooAnimal{
    public:
        // hide zooAnimal::feeding_time but not overload
        Time feeding_time(int);// 派生类成员函数
};
Bear winnie;
winnie.feeding_time("winnie");// wrong! 找不到feeding_time(string)的定义 
```
可以使用using声明的方式使得上述派生类和基类的同名成员函数构成重载关系：
```c++
class Bear: public zooAnimal{
    public:
        // now zooAnimal::feeding_time is overload with func below
        using zooAnimal::feeding_time;// 在派生类中 将基类同名函数引入
        Time feeding_time(int);// 声明派生类的同名函数
};
Bear winnie;
winnie.feeding_time("winnie");// right! 重写的Bear定义允许基类成员函数构成重载关系
```
但是，在**利用更加复杂的多继承类层次结构中的同名函数构建候选函数集时，成员函数的声明必须在同一个基类中被找到**，如果在不同基类中找到函数声明，即使它们拥有不同的参数表，也会产生重载函数调用的二义性错误：
```c++
class Endangered{
    public:
        ostream& print(ostream&);// no.1 重载print 
};
class Bear: public zooAnimal{
    public:
        using zooAnimal::feeding_time;// no.1 重载feeding_time 
        Time feeding_time(int);// no.2 重载feeding_time
        void print();// no.2 重载print
};
class Panda: public Bear, public Endangered{
    public:
        ...
};
int main(){
    Panda huanhuan;
    // wrong! 调用产生二义性 Bear::print() or Endangered::print() 候选函数列表为空
    // 解决方案 为Panda类定义一个print函数 则huanhuan.print直接调用Panda::print
    huanhuan.print();
    // right! 调用不会产生二义性 候选函数来自同一个基类(Bear) 
    huanhuan.feeding_time(56);// 候选函数集包含Bear zooAnimal两个类中的函数
}
```
### 确定可行函数
类层次结构中的继承机制会影响可行函数的确定。继承机制在确定可行函数时，允许编译器考虑一个**更大**的函数集合。确定可行函数的重点是关注参数类型转换，可以分成如下两部分进行分析：<br>
**[1] 由类中定义的转换函数定义的类型转换** <br>
```c++
class zooAnimal{
    public:
        operator const char*();// no.1 zooAnimal -> const char*
};
class Bear: public zooAnimal{...}// no.2 Bear类从zooAnimal中继承了类型转换函数
extern void display(const char*);
Bear qiuqiu;
dispaly(qiuqiu);// no.3 隐式的发生了 Bear -> zooAnimal 的类型转换
```
**[2] 由单参数非显式构造函数定义的类型转换** <br>
```c++
class zooAnimal{
    public:
        zooAnimal(int);// no.1 单参数非显式ctor int -> zooAnimal 的类型转换
};
class Bear: public zooAnimal{...}// no.2 构造函数不能被继承 无法完成 int->Bear
const int cagenum = 128;
void mumble(const Bear&);
mumble(cagenum);// 不能调用zooAnimal(int) 只能在Bear中的构造函数中查找转换函数
```
### 确定最佳可行函数
继承机制影响最佳可行函数的选择。为了选择最佳可行函数，需要将所有可行的类型转换划分等级，类层次继承关系通过影响类型转化的等级从而影响最佳可行函数的选择。<br>
**[1] 标准转换序列优先级高于用户定义的转换序列** <br>
❮标准转化序列(upcast - 可以隐式发生)❯ <br>
• 将派生类型的实参转化成任意一个基类类型的参数 <br>
• 将派生类型的指针转化成任意一个基类类型的指针 <br> 
• 将派生类型的左值转化成基类类型的引用 <br>
• c++ primer 3rd chapter 9.3.3 中给出的其他标准转化序列 - [\[Link\]](/cpp/2021/05/11/post-cpp-overload/#重载解析中第三步-find-the-best可行函数将转化序列划分等级)  <br>
❮用户定义的转化序列❯ <br>
• 类定义中的类型转化函数 <br>
• 类定义中的单参数非显式构造函数
```c++
class zooAnimal{
    public:
        operator const char*();// 用户定义的类型转化函数 
};
class Bear: public zooAnimal{...};
class Panda: public Bear, public Endangered{
    ... // 继承了zooAnimal::operator const char*() 类型转化函数
};
extern void release(const zooAnimal&);// no.1 重载函数
extern void release(const char*);// no.2 重载函数
Panda huanhuan;
release(huanhuan);// 调用标准转化no.1 而不是用户定义的no.2
```

**[2] 标准转化序列中从派生类到不同的基类类型转化(upcast)的等级划分时，距离越近优先级越高** <br>
• 转化距离和优先级关系用于左值转化成引用。
```c++
extern void release(const zooAnimal&);// no.1
extern void release(const Bear&);// no.2
Panda huanhuan;
release(huanhuan);// 调用no.2 Panda->Bear 距离较近
```
• 转化距离和优先级的关系用于指针。
```c++
void release(const zooAnimal*);// no.1
void release(const Bear*);// no.2
void release(void*);// no.3
Panda *ptr = &huanhuan;
release(ptr);// 调用优先级为 no.2 > no.1 > no.3
```

**[补充-2] 类层次结构中多继承可能导致二义性调用的问题：如果派生类到不同的基类距离相同，则调用引发二义性。**
```c++
extern void mumble(const Bear&);
extern void mumble(const Endangered&);
Panda huanhuan;
mumble(huanhuan);// wrong! 二义性的调用! Panda->Bear? or Panda->Endangered?
// 一种解决二义性调用的方案 - 使用强制类型转换
mumble(static_cast<Bear>(huanhaun));
```

**[3] 基类类型转化成派生类类型(downcast)** <br>
• 将基类类型的实参转化成任意一个派生类类型的参数 <br>
• 将基类型的指针转化成任意一个派生类类型的指针 <br> 
• 将基类型的左值转化成派生类类型的引用 <br>
```c++
extern void release(const Bear&);
extern void release(const Panda&);
zooAmimal obj;
release(obj);// wrong! 没有匹配的函数(找不到可行函数)
// 解决办法 使用dynamic_cast进行显示转化 精确匹配后调用 
release(dynamic_cast<Bear>(obj));
```

**[一个稍微特殊的例子]**
```c++
class zooAnimal{
    public:
        operator const char*();
};
extern void release(const char*);// no.1 用户定义的转化
extern void release(const Bear&);// no.2 downcast无隐式类型转化 不是可行函数 
zooAnimal obj;
release(obj);// 调用no.1 zooAnimal->const char* 看不到候选函数no.2
// 强行调用no.2的方式是 使用dynamic_cast进行类型转化 精确匹配
release(dynamic_cast<Bear>(obj));// 调用no.2
```



## Reference
> cpp primer 3rd chapter 19 p835 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式: <br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色
