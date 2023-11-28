---
layout: post
title: "return local arr from function (c++)"
author: "melon"
date: 2023-10-13 19:27
categories: "2023"
tags:
  - cpp
---

### # problematic code
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int* returnlocal(){                    // problematic code
    int arr[5] = {1, 2, 3, 4, 5};      // a local arr on stack with automatic lifetime (inside func block)
    return arr;                        // return ptr to local arr but the mem is freed
}

int main(){
    int* arr = fun();                  // problematic code:
    arr[2] = 3;                        // address of local variable returned
    cout << arr[2] << endl;            // risky action: undefined behavior
    
}
```

<hr>

### # solution 1: return local arr by static mem
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int& returnstatic(){
    static int staticvar = 10;  // a static var on static area with lifetime start here till the end of program
    return staticvar;
}

int main(){
    int& tmp = returnstatic();
    cout << "value of static var: " << tmp << endl;
    tmp += 10;
    cout << "updated value of static var: " << tmp << endl;
}
```

<hr>

### # solution 2: return local arr by heap mem
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

int* returndynamic(int size){         // dynamic arr on heap with lifetime start here till the matched delete/delete[]
    int* dynamicarr = new int[size];
    for(int i = 0; i < size; ++i){
        dynamicarr[i] = i * 2;
    }
    return dynamicarr;
}

int main(){
    int* darr = returndynamic(5);
    cout << darr[2] << endl;
    delete[] darr;                    // release mem or will mem leak
}
```

<hr>

### # solution 3: return local arr by struct wrapper
```text
///usr/bin/env make ${0%%.cpp} CXXFLAGS="-g -Wall -Wextra -std=c++11 -O1" && exec ./${0%%.cpp}
#include <bits/stdc++.h>
using namespace std;

struct arrwrapper{                     // define a struct arrwrapper
    int arr[5];
};

struct arrwrapper returnwrapperarr(){  // return a arr like a single value using deep copy
    struct arrwrapper wrapper;         // init a local struct arrwrapper for return arr
    wrapper.arr[0] = 10;
    wrapper.arr[1] = 20;
    return wrapper;
}

int main(){
    struct arrwrapper wraparr = returnwrapperarr()
    cout << wraparr.arr[0] << " " << wraparr.arr[1] << endl;
    return 0;
}
```
