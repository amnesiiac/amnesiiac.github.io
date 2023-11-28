---
layout: post
title: "function (shell)"
author: "twistfatezz"
date: 2023-04-03 20:50
categories: "2023"
tags:
  - shell
---

### # rules
$1 the space between function name && { should not be neglected
```shell
function right {
    echo "func with space ahead of {"
}
right
```
```txt
func with space ahead of {
```
the following is a wrong way to invoke shell function:
```shell
function wrong{
    echo "func without space ahead of {"
}
wrong
```
```txt
./1.sh: line xx: syntax error near unexpected token `echo "func without space ahead of {"'
```

<hr>

### # samples
shell function with params
```shell
function sum {
    echo "the result is: $(($1+$2))"  # $n: the n-th params of sum function
}
sum 3 4
```
```txt
the result is: 7
```
