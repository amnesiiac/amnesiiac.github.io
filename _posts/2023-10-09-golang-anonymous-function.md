---
layout: post
title: "go anonymous function (golang)"
author: "melon"
date: 2023-10-09 20:58
categories: "2023"
tags:
  - golang
  - ongoing
---

### # introduction
The anonymous function inside golang is bit like the "#define do(func)while(0)" trick, which will be expanded inline when executed.

<hr>

### # anonymous function
In this case, the lifecycle of var x is not determined by func square, the reference of x can be accessed by its anonymous func (stored in f).
```text
package main                 // test return anonymous func
import (
    "fmt"
)

func squares() func() int {  // return a anonymous func, the func return a int var
    var x int                // default init as 0
    return func() int {      // the function can access the var inside squares func
        x++
        return x * x
    }
}

func main() {
    f := squares()           // variable f hold the return type of squares(), thus f=func int
    fmt.Println(f())         // 1: call anonymous func for 1st time
    fmt.Println(f())         // 4: call anonymous func the 2nd time
    fmt.Println(f())         // 9: ...
    fmt.Println(f())         // 16: ...
}
```
