---
layout: post
title: "ANSI color pallete (terminal)"
author: "twistfatezz"
date: 2023-05-30 08:31
categories: "2023"
tags:
  - terminal
---

### # python script to show ANSI colors
```python
#!/usr/bin/env python3

def print_color_palette():
    for i in range(16):
        print(f"\033[48;5;{i}m {i:2d} \033[0m", end="")
    print("\n")
    for i in range(16, 232):
        print(f"\033[48;5;{i}m {i:3d} \033[0m", end="")
        if (i - 15) % 36 == 0:
            print("\n")
    print("\n")
    for i in range(232, 256):
        print(f"\033[48;5;{i}m {i:3d} \033[0m", end="")
        if (i - 231) % 6 == 0:
            print("\n")

print_color_palette()
```

<hr>

### # shell script to show ANSI colors
```shell
#!/bin/bash

print_color_palette() {
    # system colors
    for ((i = 0; i < 8; i++)); do
        echo -en "\033[48;5;${i}m ${i} \033[0m"
    done
    echo
    for ((i = 8; i < 16; i++)); do
        echo -en "\033[48;5;${i}m ${i} \033[0m"
    done
    echo -e "\n"

    # color cube (6x6x6)
    for ((i = 16; i < 232; i++)); do
        echo -en "\033[48;5;${i}m ${i} \033[0m"
        if (( (i - 15) % 36 == 0 )); then
          echo
        fi
    done
    echo -e "\n"

    # grayscale colors
    for ((i = 232; i < 256; i++)); do
        echo -en "\033[48;5;${i}m ${i} \033[0m"
    if (( (i - 231) % 6 == 0 )); then
      echo
    fi
    done
}

print_color_palette
```
