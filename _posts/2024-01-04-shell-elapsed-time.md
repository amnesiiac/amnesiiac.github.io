---
layout: post
title: "elapsed time for shell functions (shell)"
author: "melon"
date: 2024-01-04 21:04
categories: "2024"
tags:
  - shell
---

compute function running time in second unit:
```text
#!/bin/sh

do_something() {
    :
}

start_time=$(date +%s)
do_something
end_time=$(date +%s)

elapsed_time=$((end_time - start_time))
echo "script execution time: $elapsed_time s"
```

compute function running time in milisecond unit:
```text
#!/bin/sh

do_something() {
    :
}

start_time=$(date +%s%N)
do_something
end_time=$(date +%s%N)

elapsed_time=$(((end_time - start_time)/1000000))
echo "script execution time: $elapsed_time ms"
```
