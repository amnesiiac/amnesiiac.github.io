---
layout: post
title: "until: run with retry (shell)"
author: "melon"
date: 2023-12-08 22:50
categories: "2023"
tags:
  - shell
---

### # code 
below is a toy code for buildup the basic for run shell cmd with retry feature.
```text
#!/bin/sh

# set -x

retries=3
command="ls test.shh"   # string shell cmd to be executed

until $command; do
    retries=$((retries - 1))
    if [ $retries -eq 0 ]; then
        echo "cmd finished within ${retries} attemps"
        exit 1
    fi
    echo "cmd failed, retries left: $retries"
    sleep 2
done
```

<hr>

### # code in action
inside /rebornlinux/devkit/script/boostrap.sh for build up basic packages for cross-compilation toolchain.
the bootstrap.sh use /alpine/abuild/script/bootstrap.sh as the kernel.
