---
layout: post
title: "shell script dirname & filename & workdir & absolute path (shell)"
author: "melon"
date: 2024-01-04 22:04
categories: "2024"
tags:
  - shell
---

the info of a shell script mentioned in title can be derived by:
```text
#!/bin/sh
echo "basename: [$(basename "$0")]"             # the filename of the script
echo "dirname : [$(dirname "$0")]"              # the dirname when running the script
echo "pwd     : [$(pwd)]"                       # the working dir
echo "realpath: [$(realpath $(dirname "$0"))]"  # the absolute dir the script lies in
```
