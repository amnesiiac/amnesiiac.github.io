---
layout: post
title: "echo & printf (shell output)"
author: "melon"
date: 2023-07-04 08:03
categories: "2023"
tags:
  - shell
---

### # echo
todo

<hr>

### # printf
given a decimal number 8, the following printf will convert it to 2 digit hex: 
```shell
num=8
num=$(printf "%02x\n" "$num")
echo $num
```
which will output:
```txt
08
```
this is usefull when we construct mac address.
