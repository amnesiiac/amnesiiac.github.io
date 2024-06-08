---
layout: post
title: "paramter list \"$@\" (shell)"
author: "melon"
date: 2024-01-06 20:14
categories: "2024"
tags:
  - shell
---

a simple shell script to illustrate how to operate on script/function parameter list:
```text
#!/bin/sh

t() {
    echo "the global variable var: $var"               # hello
    echo "the parameter list of current function: $@"  #
}

tt() {
    echo "the global variable var: $var"               # hello
    echo "the parameter list of current function: $@"  # hello world of war
}

ttt() {
    echo "the global variable var: $var"               # hello
    set -- "${var[@]}"                                 # setup param list of cur func as "$@" by middleman $var
    echo "the parameter list of current function: $@"  # hello world
}

var=("$@")
t
tt hello world fuck
ttt
echo "$@"
```

execute the script, the output is commented inline:
```text
$ ./test.sh hello world
```
