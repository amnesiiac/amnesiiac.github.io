---
layout: post
title: "shift usages (shell)"
author: "melon"
date: 2023-12-13 22:15
categories: "2023"
tags:
  - shell
---

### # sample code for shift
```text
#!/bin/bash

echo "before shift, parameter passed: $@"
shift
echo "after shift, parameters left: $@"
```
the output is as:
```text
$ ./test.sh a b c
before shift, parameter passed: a b c
after shift, parameters left: b c
```
