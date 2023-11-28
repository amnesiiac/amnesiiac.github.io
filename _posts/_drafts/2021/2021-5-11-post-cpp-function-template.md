---
layout: post
title: "cpp - function template"
subtitle: '函数模版相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-11 22:50
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---
## 函数模版
### 1为什么要使用函数模版
函数模版提供了一种自动生成各种类型的函数实例的算法。同一函数模版生成的实例中，只有类型不同，函数参数标中的参数数量、函数体完全相同。相比函数重载机制，函数模版函数体形式固定，更适用于解决单一的只是类型不同的问题；函数重载机制则更加灵活，但是相对难以控制，更多关于函数重载，可见[这篇博客](/cpp/2021/05/11/post-cpp-overload/)。相比define定义的宏替换，使用函数模版可以避免很多错误，一种常见的错误如下所示：
```c++
#include <iostream>
using namespace std;

#define min1(a,b) ((a)<(b)? (a):(b))// 宏定义
int min2(int a, int b){
    if(a>b){return b;}
    else{return a;}
}
int p=0, pp=0, q=0, qq=0;// 虽然全局变量自动赋予初始值0 这里还是显式初始化了
int main(){
    while(min(p++,10)!=10){// 本意是想循环10次 pp=10 但是事与愿违
    	  	pp++;
    }
    std::cout<<pp<<std::endl;// 5
    while(min2(q++,10)!=10){// 调用相关函数 得到想要的结果
    	qq++;
    }
    std::cout<<qq<<std::endl;// 10
    return 0;
}
```
并且，effective c++书中指明了一条规则，尽量使用编译器代替预处理器，代码上如果使用宏定义变量，则尽可能使用const、enum、inline而不是#define。在工程代码中由于宏替换产生的错误往往令人费解，并且不容易找到错误产生的位置。

### 2函数模版应该如何进行定义 
模版参数类型分为两种：模版类型参数和模版非类型参数。模版类型参数T会被内置类型或者用户定义的类型替代，而模版非类型参数size会被各种常量值替代；这种替代的过程称之为函数模版的实例化。
```c++
template<typename T> T min(T a, T b){// 模版类型参数T
    return a<b? a:b;
}
template<typename T, int size> void equal(T a){// 模版类型参数T 模版非类型参数size
    vector<T>vec;
    for(int i=0; i<size; i++){
        vec.push_back(i);
    }
}
```
当模版类型参数名和上层域的声明的类型冲突时，函数中模版类型名将外部作用域的名字隐藏：
```c++
typedef double T;// 全局域中定义T类型
template<typename T> T min(T a, T b){// 在函数min中 全局域中的double被隐藏
    return a<b? a:b;// 返回模版类型T 而不是double类型
}
```
在模版函数体内部声明的类型，不能和模版类型名字重复，否则编译出错：
```c++
template<typename T> T min(T a, T b){
    typedef double T;// 错误! 函数内部定义的类型名T和模版类型名冲突 
    return a<b? a:b;
}
```
在模版函数参数列表中，模版类型不能重复声明(定义类型时typename和class等价)：
```c++
template<typename T, class T> T min(T a, T b){...}// 错误 模版函数类型声明重复
```
为什么需要typename？typename显式声明的其他用法：
```c++
class mytype{...};
template<typename mytype, typename T> mytype minus(mytype *arr, T val){
    mytype::name *p;// 在模版没有实例化之前 编译器不知道这是一个指针声明还是乘法操作
}
```
上述例子中产生歧义的根源在于，编译器在mytype类定义没有被实例化之前不能知道mytype::name表示什么，即编译器此时不能判断mytype类型的成员name是否是一个类型。如果name是一个类型则表示一个指针定义，否则可能被编译器推断成乘法操作。<br>
为了避免模版类型定义时产生的二义性，可以采用下面的方式进行：
```c++
class mytype{...};
template<typename mytype, typename T> mytype minus(mytype *arr, T val){
    typename mytype::name *p;// 显式的表示mytype::name是一个类型 整体是一个指针定义
}
```

### 3函数模版实例化
函数模版的实例化过程：在函数模版调用或者取函数模版地址的操作中，根据调用者的提供的模版类型参数以及模版非类型参数，经过模版实参推演构造出的独立的函数的过程。这个过程在函数模版调用或者取函数模版地址过程中隐式的发生。

##### 3.1函数模版实例化 - 通过函数调用实例化
```c++
template<typename T, int size> T min(T (&ref_arr)[size]){
    // 得到一个元素类型为T的arr的最小值
    return min_val;
}
int arr1[]={0, 1, 2, 3};// 未显式指定模版非类型参数size
double arr2[4]={0.0, 1.1, 2.2, 3.3};// 显式的指定模版非类型参数size=4
int main(){
    int i=min(arr1);// 函数模版实例化1
    double ii=min(arr2);// 函数模版实例化2
    return 0;
}
```
上面的代码中，函数模版min被实例化了两次，一次没有显式指定模版非类型参数，一次显式的制定了函数模版的全部参数。在两次实例化的过程中，通过实际调用者提供的类型参数以及非类型参数对于函数模版中的未定参数类型、数值进行推断的过程称为**模版实参推演**(在第4部分简析)。

##### 3.2函数模版实例化 - 通过函数指针进行实例化
```c++
template<typename T, int size> T min(T (&ref_arr)[size]){...}
int (*func_ptr)(int (&)[10]);// 声明一个函数指针 no.1
int (*func_ptr)(int (&)[10]) = &min;// 声明一个函数指针 no.2
```
上面代码通过声明、定义函数指针的方式进行模版实例化，方法1和方法2进行了相同的模版实例化：给予T类型参数int类型，给予非类型参数size以10的数值。

##### 3.3函数模版实例化 - 无法确定使用哪个版本重载函数进行实例化
```c++
template<typename T, int size> T min(T (&)[10]){...}// 函数模版的定义
typedef int (&ref_arr1)[10];// 数组引用类型声明1
typedef double (&ref_arr2)[20];// 数组引用类型声明2
void func(int (*)(ref_arr1));// 声明了重载函数1 参数为数组的指针
void func(double (*)(ref_arr2));// 声明了重载函数2 参数为数组的指针
int main(){
    func(&min);// 错误! 无法确定func使用重载函数的那个版本 从而不能进行函数模版推断
}
```
上述例子中func函数被重载，通过调用func(&min)的方式无法确定使用重载函数的那个版本对于min参数进行实例化，即不知道传递给模版函数min的T类型参数是int还是double，因此引发编译错误。

##### 3.4 函数模版实例化过程 - 强制类型转化以确定使用哪个重载函数实例化
```c++
int main(){
    func(static_cast<double (*)()> (&min));// 编译时正确 运行时模版实例化可能会出错 
}
```
上面的代码至少在编译时，是没有问题的：使用了强制类型转化，编译器通过类型转化能够唯一确定使用的重载函数版本，因此，能够进入正常的模版函数推演。例子中func的参数类型是数组的指针类型，而传入的是一个函数的引用(通常用来初始化函数指针)，在模版函数推演的过程可能会报错，但是这已经是运行时模版实例化的问题了，在编译时不会报错。

### 4模版实参推演
当函数模板被调用时，对函数实参类型的检查决定了模板实参的类型和值，这个过程被称为模板实参推演(template argument deduction)。
函数模版实参推演必须要能严格匹配或者能够通过允许的三种转换方式进行转化后匹配。模版类型推演允许的转化有：左值转换、限定修饰符转换和派生类类型到一个基类类型的转换(该基类根据一个类模板实例化而来)。
##### 4.1模版实参推演 - 严格匹配的转化 & 不能匹配的转化
```c++
template<typename T, int size> T min(T (&ref_arr)[size]){...}
// 能够匹配的情况
void func(){
    double arr[8]={10.3, 7.2, 14.0, 3.8, 25.7, 6.4, 5.5, 16.8};
    int = min(arr);// 正确! 模版函数推演匹配与否只考虑参数表 不考虑返回类型
}
// 不能匹配的情况 
void func(int arr[9]){
    // 错误! arr类型是int*[9] 而模版函数需要的参数类型为数组类型左值T (&)[size]
    int val=min(arr);
}
```
##### 4.2模版实参推演 - 4种左值转化
模版实参推演 - 通过左值转化之后进行匹配。左值转化包括：左值到右值的转换、从数组到指针的转换或从函数到指针的转换。下面给出一个从数组名到指针转化的例子：
```c++
template<typename T> T min(T *arr, int size){...}
int arr[4]={0, 1, 2, 3};
int main(){
    int size=sizeof(arr)/sizeof(arr[0]);
    min(arr, size);// 从指向4个int元素的数组名类型 隐式的转化成 T*指针类型
}
```
##### 4.3模版实参推演 - 限定修饰符转化
模版实参推演 - 限定修饰符转化：给指针添加const和volatile关键字，注意反过来不能转化(const和volatile限定符不能消除)。具体的，通过下面的例子进行理解：
```c++
template<typename T> T min(const T *, int size){...}
int arr[4]={0, 1, 2, 3};  int *ptr=&arr[0];// ptr是指向数组首元素的指针
int main(){
    int size=sizeof(arr)/sizeof(arr[0]);
    min(ptr, size);// 从int *类型通过限定修饰符转化成 const int *类型
}
```
##### 4.4模版实参推演 - 从派生类到基类的类型转化
模版实参推演 - 到一个基类的转化：如果函数参数的类型是一个基类模板类型，且实参是这个基类模版的派生类模版类型，则模板实参的推演就可以正常进行：
```c++
// 定义了一个类模版:arr_type<T> 通过将类型T实例化可以得到一个类类型 
template<typename T> class arr_type{...}
template<typename T> T min(arr_type<T> &arr){...}
// 定义了一个派生类模版:inherit_arr_type<T> 继承自基类arr_type<T> 是一个继承类类型
template<typename T> class inherit_arr_type : public arr_type<T>{...}
int main(){
    // 创建一个继承类的实例obj_arr 并且调用构造函数对这个继承类数组obj初始化
    inherit_arr_type<int> obj_arr(arr, sizeof(arr)/sizeof(arr[0]));
    // 正确! 隐式的进行从继承类类型inherit_arr_type<int>转化成基类类型arr_type<int>
    min(obj_arr);
}
```
##### 4.5模版实参推演 - 推演的统一性要求
模版实参推演 - 当模版参数表中多个参数类型都和模版类型T相关，则每个模版实参推演出的类型都应该和第一个被推演出的类型一致，否则将推演失败。所有相同类型的模版实参不能分开考虑。简而言之：推断过程中模版类型要保持一致性，不参与推演的已知参数可以进行任何合理的类型转换。
```c++
// 定义了一个两个参数都和模版类型T相关的min函数
template<typename T> T min(T, T){...}
unsigned int ui;
int main(){
    // 错误: 不能实例化 min5( unsigned int, int )
    // 必须是: min(unsigned int, unsigned int) 或 min(int, int)
    min5(ui, 1024 );
}
```
##### 4.6函数模版实参推演总结
`1` 依次检查每个调用函数的每个实参，并确定调用函数的实参和模版函数参数的对应关系。<br>
`2` 对于1中找到的参数对应关系，根据上述介绍的推演规则，推演出相应的模版实参。<br>
`3` 在推演的过程中，调用函数实参和推演得到的模版函数实参不一定完全相同，只要满足调用函数实参能够通过限定好的转化规则转化成模版类型实参即可。其中转化类型包括：左值转化、限定修饰符转化、从派生类模版到基类模版的转化(函数参数表中涉及到基类类类型)。其中所有的转化规则都需要遵守：多个不同的模版实参推演的结果相同。

### 5显式声明模版实参
##### 5.1显式声明模版实参 - 结局模版参数推演二义性问题
显式声明模版实参 - 解决模版参数推演不统一的问题。<br>
函数模版参数列表中有很多相同类型的参数，它们在通过调用者进行实参推演的结果应当统一。当模版实参类型被显式的指定之后，就可以利用任何形式的实参推演了。
```c++
template<typename T> T min(T, T){...}
unsigned int ui; int i;
int main(){
    // 错误! 不能成功实例化函数模版 类型参数T贵言结果不统一 int - unsigned int
    // 正确的推演结果应该是 min(int,int) or min(unsigned int, unsigned int)
    min(ui, i);
    // 通过显式指定模版实参类型的方式 可以解决上述无法类型无法推演的问题
    // 此时 可以使用任何形式的实参推演: unsigned int -> int
    min<unsigned int>(ui, i);// 显式指定 函数实例类型 min(ui, ui)
}
```
##### 5.2显式声明实参 - 解决参数无法正常推演的问题
显式声明模版实参 - 解决函数模版无法对于返回类型进行推演的问题。<br>
假定我们想定义一个求和函数模版，模版返回类型能够容纳任意两个参数类型的数值求和的结果，初步的想法如下：
```c++
template<typename T, typename U> xxx min(T, U);// 返回类型是什么类型???
// 下面两种方法都不对 使用U或者T都可能在某个计算中导致溢出
template<typename T, typename U> U min(T, U);
template<typename T, typename U> T min(T, U);
```
我们可以尝试在函数模版中为返回函数设置一个参数类型，通过在实例化时再指定返回类型的方式解决上述问题：
```c++
// 显式在函数模版类型中指定了参数R作为带推演的返回类型
template<typename R, typename T, typename U> R min(T, U);
```
但是这样做有潜在的问题：在模版函数参数推演的过程中，我们只能推演模版参数表中的参数，对于函数返回值，不能够进行推演。对于不能推演的返回参数，使用显式声明的方式可以解决。显式声明需要按照函数模版参数表的顺序进行声明，和函数缺省参数一样，不能跳跃省略(为了只省略不能推演的参数而跳跃是不可行的)，只能省略尾部连续的参数。
```c++
template<typename R, typename T, typename U> R min(T, U);
char ch; unsigned int ui;
unsigned int result=sum(ch, ui);// 错误! 返回类型不能被推演出来
// 使用显式声明的方式 对于不能推演的实参赋予类型
unsigned int result=sum<unsigned int>(ch, ui);// 显式给出返回值的类型
// 如同缺省实参一样 只能省略尾部的实参 
unsigned int result=sum<unsigned int, char>(ch, ui);// 正确 省略尾部可推演参数
unsigned int result=sum<unsigned int, ,unsigned int>(ch, ui);// 错误!
// 显式声明模版实参的实例化的函数 用于定义函数指针
unsigned int (*func_ptr)(char, unsigned int)=&sum<unsigned int>
```
再看一个复杂一点的例子：
```c++
template<typename T, typename T, typename U> R min(T, U){...}
void manipulate(int (*func_ptr)(int, char));
void manipulate(double (*func_ptr)(float, float));
int main(){
    // manipulate是一个重载函数 因此无法确定是哪一种重载形式进行模版实例化
    // 即便确定了进行实例化的重载函数的版本 也不能确定推断出返回类型
    manipulate(&sum);// 错误
    // 通过显式声明模版实参的方式 确定了重载函数的版本 并且能够正常推演模版类型参数
    manipulate(&sum<double, float, float>);// 正确
}
```
##### 5.3显式声明模版规则
显式声明模版实参的使用规则：需要注意，虽然显式声明模版实参能够很方便的帮助我们解决很多棘手的推演失败问题，但是不要滥用这种方法。<br>
显式声明模版实参应当只被用在解决模版参数推演中产生二义性而不能推演的情况；以及其他不能推演出参数类型的情况：如不能推断出函数返回类型，或者模版实例化的函数为一组重载的函数，无法确定哪个重载版本对于函数模版进行实例化。<br>
从另一个角度看，尽可能在必要时才使用显式声明模版实参，因为在非必要情况下，让编译器帮助我们进行参数推断是很有效率的事。其次，指定了过多的非必要实参破坏了函数模版容纳新类型的能力(泛化能力)；并且，制定了过多的非必要实参，在函数接受新定义类型的数据时，需要重新检查显式声明参数类型的兼容性。

### 6模版编译模式
函数模版相当于无限个类型相关的函数实例集合的规则的一种描述，它本身并不是函数的定义。当编译器看到函数模版的定义时，只会将模版函数的表示形式记录，当编译器看到模版函数被使用时，才会为其生成一个相应类型的定义。<br>
有两种声明定义函数模版的方式可供选择：**1** 把函数模版声明定义放在头文件中(类似于inline函数的声明定义方法)，然后在使用函数模版实例的地方包含该头文件；**2** 或者只在头文件中给出函数模版的声明(类似于非inline函数声明定义方法)，把函数模版定义放在源文件中。这两种方式的选择涉及c++模版编译模式(template compilation model)，分别对应包含模式(inclusion model)和分离模式(separation model)。下面分别进行简述：

##### 6.1包含编译模式
包含编译模式下：在每个模版被实例化的地方包含函数模版的定义，通常这种模式的函数模版定义在头文件中，声明定义方式类似inline函数。
```c++
// header_file.h
template<typename T> T min(T t1, T t2){// 函数模版的声明定义
    return t1<t2? t1:t2;
}
// source & user file: user.c
#include "header_file.h" // 在每个函数模版被实例化的地方包含该头文件
int i, j; double result=min(i, j);
```
实际工程文件中，包含编译模式下的函数模版所在的头文件通常被多个源文件所包含，编译器不会为每个源文件中的模版调用都实例化一次，但是编译器会为每个类型的模版调用进行实例化，min(int, int)只被编译器实例化一次，其他相同的调用则直接进行实例引用。这种优化的实例化具体在何时何地进行取决于编译器的具体实现，可查阅相关书籍文档进行了解。

在头文件中声明编译函数模版的缺点：**1** 头文件中的函数模版实现内容曝露给用户：用户可能不关心函数模版的具体实现；程序作者想要对用户隐藏具体函数实现。**2** 如果函数模版定义非常大，在头文件中给出太多的细节实现可能是不被接受的。并且在多个文件中编译相同的函数模版增加了编译的时间。

##### 6.2分离编译模式
分离编译模式：函数模版的声明放在头文件中，函数模版的定义放在源文件中。这种声明定义的方式类似一个非inline函数的形式。
```c++
// header_file.h
template<typename T> T min(T, T);// 函数模版的声明
// source file: header_file.c
export template<typename T> T min(T t1, T t2){...}// 函数模版的定义 
// user file: user.c
#include "header_file.h"// 只需要包含声明头文件即可
int i, j; double result=min(i, j);
```
这种情况下，即使函数模版声明的源文件对于用户代码不可见，仍然可以让用户调用相关的函数模版进行实例化。

**export关键字**<br>
在源文件中的**export**关键字将min函数模版定义成一个可导出的模版(exported)；export告诉编译器在其他文件中的函数实例中可能会用到这个模版定义，即保证了生成其他文件的函数模版实例时，这个定义对编译器是可见的。当函数模版定义成可导出时，我们可以在任意程序文件中实例化该函数模版定义，如果export关键字被省略，编译器可能不能正确实例化模版，从而导致链接失败。export关键字在声明中是可选的，只需在定义中添加export即可。<br>
需要注意的是：一个函数模版在程序中只能定义为export一次，函数模版在多个文件(不同的编译单元)中被定义成export不能被编译器发现(编译器每次只处理一个编译单元)，但是可能引发下面的错误: **1** 产生链接错误，指出函数模版在多个文件中被定义。**2** 编译器可能会为相同类型的函数模版实例化不止一次，由于模版存在多个相同的实例，导致链接错误。**3** 编译器可能会以其中一个export函数模版进行实例化，忽略其他定义。<br>
更多关于*export*关键字相关知识整理，详见[博客](/cpp/2021/06/10/post-cpp-export/)中的内容。

最后，总结关于分离编译模式的内容。分离模式能够很好的将函数声明、实现(定义)以及使用三种情景相互分开，可以使得程序的组织比较有逻辑性。但是不是所有的编译器都支持分离编译模式，部分编译器对于分离编译模式不能很好的支持，这是因为分离编译模式需要较为复杂的编译环境。

##### 6.3显式实例化声明
早期的编译器在实例化函数模版的时候，常常发生各种问题：同类型的函数模版被重复实例化很多次，影响了编译的效率；如果程序由很多文件组成，文件中的模版都被实例化，那么编译应用程序需要的时间将会显著的增加。通过显式实例化声明，可以让程序员控制编译器生成这些实例的时间，从而一定程度上避免了无用的实例化。
```c++
template<typename T> T sum(T t1, int t2){...};// 函数模版的定义
template int * sum<int*>(int *, int);// 显式实例化声明 要求使用int*初始化T
```
需要注意的是，显式实例化声明在程序中只能出现一次；并且在显式实例化声明出，函数模版的定义必须可见，否则导致编译错误。下面的例子中，显式实例化声明看不到函数模版的定义，编译错误：
```c++
template<typename T> T sum(T t1, int t2);// 函数模版的声明
template vector<int> sum<vector<int>>(vector<int>, int);// 显式实例化声明失效
```
在实际使用过程中，显式实例化声明是和特定的编译选项联合使用的：显式实例化声明和编译器中压制模版隐式实例化选项联合使用，在不同的编译器中，这个编译选项可能有所差别。使用了压制模版隐式实例化之后，编译器不会自动的为模版提供隐式实例化，从而需要程序员在需要使用实例化函数的地方，进行显式的实例化声明后再使用相应函数。

### 7模版显式特化
前面的内容介绍了模版实例化的具体规则和使用方法，实例化只能在模版类型实例推演中做出实例声明；不同于函数模版实例化，特化的含义是，不仅仅是类型特化，而且包含了函数具体内部实现的特化。**简而言之：实例化操作的是函数模版的类型，而特化操作的是函数模版的类型和函数体**。
##### 7.1为什么需要模版显式特化
在编写函数模版时，我们并不能保证写出针对所有可能的实例的实现都非常适合的函数模版，因此在某些情况下，需要我们通过显示特化的方式来写出比实例化更加高效的函数。
```c++
template<typename T> T max(T t1, T t2){
    return (t1>t2? t1:t2);
}
```
如果函数模版通过类型const char *对于上述函数模版直接进行实例化，那么参与返回类型比较的是字符串的指针而不是字符串本身，此时使用通用的函数模版得不到我们希望的结果。这种通用函数模版无法实现的功能需要借助函数模版显示特化来实现。
```c++
// 针对min(const char*, const char *)进行模版特化 -> 重写函数实现
template<> const char* max<const char*>(const char *s1, const char*s2){
    return (strcmp(s1,s2)>0? s1:s2);// 通过函数模版特化进行重新定义
}
```
由于有了上述的特化的函数模版实现，在调用特定类型的min函数时，会直接调用特化的实现而不是通用的实现。
```c++
int main(){
    int i=min(10, 5);// 调用通用的函数模版
    const char *p = min("hello", "world");// 调用const char*函数特化模版
}
```
##### 7.2模版显式特化的声明
下面三种函数模版显式特化声明都是正确的，第一种是特化声明的标准形式。当模版实参可以从函数模版参数列表中推断出来，则模版实参可以省略：见第二种特化声明no.2。另外，第三种写法no.3也是正确的，但是它不是一个模版函数，相当于一个普通函数的声明，因为声明和模版函数相匹配的的普通函数并不是个错误。
```c++
template<> const char* min<const char*>(const char*, const char*);// no.1
template<> const char* min(const char*, const char*);// no.2 省略模版实参
const char* min(const char*, const char*);// no.3 省略template<>的普通声明
```
第一种方式和第二种方式的区别是：能够将模版实参从函数模版参数列表中推演出来。为什么需要第三种常规函数声明方式呢？因为，前两种模版函数参数推演的过程只能从**适用于模版实参推演**的类型转化中完成，第三种常规函数声明不受到这些限制，只要是编译器允许的类型转化都可以进行。


### 8重载函数模版 
函数可以重载以适应相同函数名，不同实现的任务，同样，函数模版已可以被重载。模版主要的功能是类型特化(可以接受部分定义特化)，重载函数主要的功能是定义特化(可以接受部分的类型特化)。两者的功能不同但又略有有兼容，因此，将它们结合在一个构成的重载函数模版是一个非常强大的编程范式 - 我全都要的逻辑。下面展示了如何重载函数模版，并进行使用：
```c++
template<typename T> class arr{...};
template<typename T> T min(const arr[T]&, int);// no.1
template<typename T> T min(const T*, int);// no.2
template<typename T> T min(T, T);// no.3
int main(){
    arr<int> obj1(1024);
    int obj2[1024];
    int val1 = min(obj1, 1024);// no.1 类型匹配
    int val2 = min(obj2, 1024);// no.2 类型匹配
    double val3 = min(obj1[0], obj2[0]);// no.3 需要类型转化
}
```
##### 8.1重载过程的二义性
重载函数模版之后，需要保证具体函数实例调用时，不产生二义性(不知道调用哪个形式进行实例化)。
```c++
template<typename T> int min(T, T){...};// 函数模版的定义
int i; unsigned int ui;
min(1024, i);// 正确! 将模版类型推演成int
min(i, ui);// 错误! 模版实参推演是分别进行的 两个参数推演成不同的类型
```
上述函数模版的实例化中，出现了同一个参数实例化的二义性，参数推演失败。我们可以补充一个函数模版的声明，使其能够正确实例化。
```c++
template<typename T> int min(T, T);// 函数模版声明1
template<typename T, typename U> int min(T, U);//函数模版声明2
min(i, ui);// 正确! 可以正确调用第二个声明方式
min(1024, i);// 错误! 无法确定使用哪个声明进行实例化
```
上述函数模版的调用在第二个实例化中又产生了二义性，因为模版声明2声明的类型T和U可以接受不同类型也可接受相同类型。此时min(1024,i)调用哪一个模版都是可行的。解决调用哪个函数模版，并且消除调用时的二义性的**唯一方法**是**显式声明模版实参**。
```c++
min<int, int>(1024, i);// 显式声明模版实参 只能调用第二个函数模版
```
但是，存在特殊情况：即使一个函数调用能够同时被两个函数模版实例化，但这个函数调用时仍然不是二义的。如下面代码所示：
```c++
template<typename T> T sum(T, int);// no.1声明
template<typename T> T sum(T*, int);// no.1声明的一个更特化的声明 
int i[1024];
int val = sum<int>(i, 1024);// 正确! 这是一个没有二义性的调用
```
上述代码没有产生二义性调用的原因是：第二个声明是sum函数的**更特化调用**，因此在调用的过程中，会选择更特化的调用而忽略no.1声明。<br>
**什么是更特化的声明?** 一个函数模版比另一个函数模版更特化：两个函数模版必须具有相同的名字，相同的参数个数，对于两者不同类型的参数(T&T*)，一般的模版接受的参数类型返回是特化的模版接受的参数类型范围的超集。

### 9函数模版的重载解析
同函数重载解析一样，函数模版的重载也许需要通过编译器解析之后，才能决定具体实例化or特化的调用方式。解析的最终目的是：**找到一个函数模版的声明定义，使其和函数调用者构成最佳的匹配关系。**<br>
**[1] 解析第一步：构造可以被调用的候选函数集** <br>
考虑与函数调用同名的函数模板：如果对于该函数调用的实参，模板实参推演能够成功，则**实例化**一个函数模板(得到一个实例化函数)。或者对于推演出来的模板实参存在一个模板特化，则该**模板特化**就是一个候选函数(得到一个特化函数)。

**[2] 解析第二步：从候选函数集中选出可行函数集** <br>
这部分内容可以参考[博客](/cpp/2021/05/11/post-cpp-overload/)中整理的内容进行理解。

**[3] 解析第三步：将第二步所做的类型转化划分等级，越是精确匹配的转化，被最终调用的优先级越高。** <br>同样根据[博客](/cpp/2021/05/11/post-cpp-overload/)中的整理，可以简单介绍相关的优先级如下：<br>
**a** 如果只选择了一个函数，则调用该函数。<br>
**b** 如果该调用是二义的，则从可行函数集中去掉函数模板实例。

**[4] 解析第四步：验证得到的结果。只考虑可行函数集中的普通函数 完成重载解析过程** <br>
**a** 如果只选择了一个函数，则调用该函数。<br>
**b** 否则，该调用是二义的。

### 10函数模版定义中的名字解析
函数模版定义中，代码可以被分成两个部分，一部分和函数模版参数类型相关，另一部分是已知的参数，和采用哪种实例化方式无关。它们在函数模版实例化中，对名字解析有不同的要求。这一部分用来将具体它们之间的区别。一个典型的函数模版程序如下：
```c++
template<typename T> T min(T *arr, int size){
    T min_val=arr[0];
    for(int i=1; i<size; i++){
        if(arr[i]<min_val){
            min_val=arr[i];
        }
    }
    print("min_val found!");// no.1 print打印字符串类型数据
    print(min_val);// no.2 print打印模版参数类型数据
    return min_val;
}
```
在获取数组arr最小值的函数min中，传入数组arr以最小值的类型都是模版参数，而数组的大小size一直都是int类型。即分别对应了依赖模版参数类型的参数、不依赖模版参数类型的参数。no.1函数任何时候都知道调用什么版本的print函数，然而no.2函数在模版的实例化之前都不能确定调用print函数的具体版本。

##### 10.1函数模版名字的正确声明方式
函数应当在使用之前被声明，因此no.1函数在函数模版定义之前被声明，而no.2函数应当在函数模版具体实例化之前被声明(采用包含编译模式组织文件)：
```c++
// -- header_file.h --
void print(const char*);// 在模版定义之前显式的声明no.1 print函数
template<typename T> T min(T *arr, int size){
    // 获得数组arr的最小值min_val
    print("min_val found!");// 函数模版定义中的no.1 print定义
    print(min_val);// 模版定义中 不能确定print调用版本
}
// -- user.cpp --
#include "header_file.h"
void print(int);// 声明使用的no.2 print的版本 - 必须在函数模版实例化之前声明
int arr[4]={0, 1, 2, 3};
int main(){
    int size=sizeof(arr)/sizeof(arr[0]);
    min(&arr, size);// 在用户文件中进行函数模版实例化
}
```
上述程序文件中，如果程序员忘记在模版定义的头文件中给出声明，他想要在用户文件中函数模版实例化之前给出一个补充声明，则这个声明是可见的但是不会被调用。<br>
另外，上述程序用户文件中，如果print(int)版本在函数模版min实例化之前没有被声明，那么将会导致编译时刻错误：未定义的函数print。<br>

根据上述程序的分析：模版中使用的名字定义需要分两个部分进行：**不依赖于模版参数的名字在函数模版定义时被解析，因此需要在函数模版定义之前被声明；依赖于模版参数的名字在函数模版实例化时被解析，因此需要在函数模版实例化之前被声明。**

**Advice for 程序员:** 对于程序设计者而言，很多时候会面对错综复杂的头文件包含关系，以及函数模版在多个用户文件中被实例化的情况，需要绘制好模版及其内部名字定义的包含关系图表，确保能够正确的组织名字定义的声明-定义关系，函数模版的声明-实例化的关系。

##### 10.2函数模版实例化点的确定
函数模版定义中和模版参数类型无关的名字需要函数模版的设计者进行声明处理，对于用户而言，如何正确使用(实例化)函数模版才是关键。函数模版类型相关的名字需要在函数模版实例化之前被声明，解决这个声明的关键是确定函数模版的具体实例化点(point of instantiation)。下面结合代码进行分析：

**[1] 函数模版的实例化点总是在名字空间中出现(不在局部域中出现) 并且紧跟在引用它的函数后面**
```c++
// -- user.cpp --
int main(){
    min(&arr, size);// 函数模版在main函数中被引用
}
int min(int *arr, int size){...}// no.1 min函数在编译器中的实例化点
```
**[2] 函数模版被多次引用时，编译器在规则[1]产生的实例化点中自由选择一个对函数模版进行实例化**
```c++
// -- user.cpp --
#include "header_file.h"
void another_init_point();
int arr[4]={0, 1, 2, 3};
int size=sizeof(arr)/sizeof(arr[0]);
int main(){
    min(&arr, size);// no.1 函数模版引用点
}
// 由于no.1引用函数模版 根据规则一产生的'候选'实例化点
void another_init_point(){
    min(&arr, size);// no.2 函数模版引用点
}
// 由于no.2引用函数模版 根据规则一产生的'候选'实例化点
```
上面的代码有2个实例化点，分别是no.1函数模版以及no.2函数模版产生的。对于由于多处引用函数模版产生的多处实例化点，编译器自由地选择其中一个进行真正的模版初始化。对于上述情况，我们可以将函数模版参数类型相关的名字声明在一个头文件(如:user.h)中，从而避免实例化找不到名字声明：
```c++
// -- user.h --
void print(int);
// -- user.cpp -- 
#include "header_file.h"
... // 函数模版引用点 no.1
... // 函数模版引用点 no.2
```
上述的例子都是在单个用户文件(user.cpp)中进行函数模版的引用，如果函数模版声明文件(header_file.h)被许多个用户文件(user1.cpp user2.cpp...)包含，那么如何确定函数模版的实例化点，又如何进行参数类型相关的名字声明呢？<br>
答：我们需要**小心地**将声明了函数模版类型相关的名字声明放在user.h中，并将user.h**小心地**包含在所有引用了该函数模版的user.cpp中。

### 名字空间和函数模版
函数模版应当定义在名字空间中，可以是全局名字空间，也可以是局部名字空间。全局名字空间模版在使用时可以省略域访问限定符，局部名字空间模版在使用时需要加上访问限定符，或者提供一个using声明。<br>
**[1] 在名字空间定义的模版的普通实例化调用流程。**
```c++
// -- header_file.h --
namespace cpp_primer{
    template<typename T> T min(T *arr, int size){...}
}
// -- user.c --
#include "header_file.h"
int arr[4]={0, 1, 2, 3};
int size=sizeof(arr)/sizeof(arr[0]);
int main(){
    min(&arr, size);// 错误! 不能直接直接引用函数模版min
    using cpp_primer::min;// 正确! 通过using引入相应声明
    min(&arr, size);
}
```
**[2] 在名字空间定义的模版的特化定义调用流程。**
```c++
// ---------- header_file.h ----------
namespace cpp_primer{
    template<typename T> T min(T *arr, int size){...}// 函数模版的声明定义
}

// ---------- user.h ----------
class SmallInt{
    public:
        SmallInt(int val):value(val){}// 构造函数
        // 友元函数声明 - 能够方便调用私有成员
        friend bool compareless(const SmallInt&, const SmallInt&);
    private:
        int value;// 私有数据成员

};
// 模版参数类型函数 - 需要定义在user.h中 
void print(const SmallInt&);
// 模版参数类型函数 - 需要定义在user.h中 - 访问私有成员需要在类中声明为友元
bool compareless(const SmallInt&, const SmallInt&);

// ---------- user.cpp ----------
#include "header_file.h"
#include "user.h"
// no.1 通过引用名字空间的方式在用户文件中给出特化定义
namespace cpp_primer{
    template<> SmallInt min(SmallInt *arr, int size){...}// 函数模版特化定义
}
// no.2 通过名字空间域访问作用符的形式给出特化定义
template<> SmallInt cpp_primer::min(SmallInt *arr, int size){...}
SmallInt arr[4];
int size=sizeof(arr)/sizeof(SmallInt);
int main(){
    // no.1 using声明引入函数模版特化
    using cpp_primer::min;// 在局部空间域中声明名字空间中的实体
    min(&arr, size);// 调用user.cpp中实现的特化定义
    // no.2 域访问作用符引入函数模版特化 - 我觉得这种方式更加清晰
    cpp_primer::min(&arr, size);
}
```

### 一个函数模版程序实例 - 泛型算法初探
这一部分我们将实现自定义的数组，以及相应自定义的数组操作；为了能够使得数组能接受多种类型，采用函数模版的方式进行定义。为了简单起见，采用包含编译模式组织代码。
```c++
// ---------- arr.h ----------
template<typename T> class Array{
    public:
        // constructor - destructor
        explicit Array(int size=DefaultArraySize);
        Array(const Array &rhs); 
        Array(T *array, int array_size);
        virtual ~Array(){delete [] ia;}
        // operator overload
        bool operator==(const Array&) const;
        bool operator!=(const Array&) const;
        Array& operator=(const Array&);
        virtual T& operator[](int index){return ia[index];}
        // tool func
        virtual sort(); 
        virtual T min() const;
        virtual T min() const;
        virtual int find(const T &val) const;
        int size() const {return _size;}
    protected:
        static const int DefaultArraySize=12;
        int _size;
        T *ia;
};
// sort函数模版 - 通过快速排序算法实现对于Array<T>类型数组的排序
template<typename T> void sort(Array<T> &arr, int low, int high){
    if(low<hight){
        int l=low;
        int h=high+1;
        T ele=Array[l];
        for(;;){
            while(min(arr[++l], ele) != ele && l<high);
            while(min(arr[--h], ele) == ele && h>low);
            if(l<h){
                swap(arr, l, h);// no.1 aux function
            }
            else{
                break;
            }
        }
        swap(arr, low, h);
        swap(arr, low, h-1);
        sort(arr, h+1, high);// no.2 aux function
    }
}
// no.1 aux function template def
template<typename T> T min(T a, T b){
    return a<b? a:b;
}
// no.2 aux function template def
template<typename T> void swap(Array[T] &arr, int i, int j){
    T tmp=arr[i];
    arr[i]=arr[j];
    arr[j]=tmp;
}
// display function template for Array
template<typename T> void display(Array<T> &arr){
    std::cout<<"<";
    for(int i=0; i<arr.size(); ++i){
        std::cout<<arr[i]<<" ";
    }
    std::cout<<">"<<std::endl;
}
// ---------- user.cpp ----------
#include <iostream>
#include <string>
#include "arr.h"
double da[10]={26.7, 5.7, 37.7, 1.7, 61.7, 11.7, 59.7, 15.7, 48.7, 19.7 };
int ia[16]={503, 87, 512, 61, 908, 170, 897, 275, 653, 426, 154, 509, 
            612, 677, 765, 703 };
string sa[11]={"a", "heavy", "snow", "was", "falling", "when",
            "they", "left", "the", "police", "station"};            
int main(){
    // construct Array
    Array<double> double_arr(da, sizeof(da)/sizeof(da[0]));
    Array<int> int_arr(ia, sizeof(ia)/sizeof(ia[0]));
    Array<string> string_arr(sa, sizeof(sa)/sizeof(sa[0]));
    // init double arr
    cout<<"sort double_arr size("<<double_arr.size()<<")"<<endl;
    sort(double_arr, 0, double_arr.size()-1);
    display(double_arr);
    // init int arr
    cout<<"sort int_arr size("<<int_arr.size()<<")"<<endl;
    sort(int_arr, 0, int_arr.size()-1);
    display(int_arr);
    // init string arr
    cout << "sort string_arr size("<<string_arr.size()<<")"<<endl;
    sort(string_arr, 0, string_arr.size()-1);
    display(string_arr);
    return 0;
}
```

## Reference
> cpp primer 3rd chapter 16.1 p407 <br>
> cpp primer 5th chapter 16.1.2 p609 

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
