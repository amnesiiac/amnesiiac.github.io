---
layout: post
title: "invoke function by name (c++)"
author: "melon"
date: 2023-10-15 12:46
categories: "2023"
tags:
  - cpp
---

the following provide a function map, in which the func name is mapped to its corresponding
class method's pointer. once the class object got instantiated, then its method can be called
by using the function map with its name key.

see the toy code following:
```text
#include <iostream>
#include <map>
#include <string>

class Foo{                                                           // class with func method
public:
    void func0(){
        std::cout << "function 0 called." << std::endl;
    }

    void func1(){
        std::cout << "function 1 called." << std::endl;
    }

    void func2(){
        std::cout << "function 2 called." << std::endl;
    }

    void func3(){
        std::cout << "function 3 called." << std::endl;
    }
};

int main() {
    std::map<std::string, void (Foo::*)()> function_map;              // map<func_name, func_pointer>

    function_map["function0"] = &Foo::func0;                          // insert map ele: <name, func_addr>
    function_map["function1"] = &Foo::func1;
    function_map["function2"] = &Foo::func2;
    function_map["function3"] = &Foo::func3;

    Foo foo;                                                          // create instance

    std::string key = "function0";                                    // invoke method by name
    (foo.*(function_map[key]))();

    for(auto it=function_map.begin(); it!=function_map.end(); ++it){  // invoke method by addr
        if(it!=function_map.begin()){
            (foo.*(it->second))();
        }
    }
    return 0;
}
```
