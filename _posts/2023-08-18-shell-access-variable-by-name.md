---
layout: post
title: "access variable by its name (shell)"
author: "melon"
date: 2023-08-18 21:59
categories: "2023"
tags:
  - shell
---

### access variables by its name: ${!patch}
```text
#!/bin/bash

OPTION1="hello-world1"
OPTION2="hello-world2"
OPTION3="hello-world3"

num=1
patch=OPTION1                               # patch hold variable name

while [ ! -z "${!patch}" ]; do              # while the variable named ${patch} is not empty
    options="$options ${!patch}"            # access variable by variable patch
    num=$((num+1)) && patch=OPTION${num}
done
echo $options
```
output:
```txt
hello-world1 hello-world2 hello-world3
```
