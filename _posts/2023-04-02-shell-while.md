---
layout: post
title: "while structure (shell)"
author: "twistfatezz"
date: 2023-04-02 18:09
categories: "2023"
tags:
  - shell
---
### samples
normal while loop
```shell
count=0
while [[ $count -lt 10 ]]; do
    count=$((count+1))  # self-add
    echo "while loop: $count"
done
```
<br>


while with break
```shell
count=0
while [[ $count -lt 5 ]]; do
    echo "current count: $count"
    count=$((count+1))
    if [ $count -eq 7 ]; then
        break;  # break for
    fi
done
```
<br>


while + continue
```shell
count=0
while [[ $count -lt 5 ]]; do
    count=$((count+1))
    if [ $(($count%2)) -eq 0 ]; then
        continue;  # skip for
    fi
    echo "current count: $count"
done
```

