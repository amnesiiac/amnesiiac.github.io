---
layout: post
title: "bsd commands vs gnu commands (shell)"
author: "twistfatezz"
date: 2023-06-22 11:12
categories: "2023"
tags:
  - shell
---

### # bsd sed vs gnu sed
linux gnu sed:
```shell
# read file & delete the last line if it is \n
sed '/^[[:blank:]]*$/{$d;}' tmp
```
macos bsd sed:  
```shell
sed -i '' '/^[[:blank:]]*$/{$d;}' tmp
```
if use gnu sed on macos, will result in:
```txt
sed: -I or -i may not be used with stdin
```

