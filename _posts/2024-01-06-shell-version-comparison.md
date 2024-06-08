---
layout: post
title: "version comparison helper (shell, utility)"
author: "melon"
date: 2024-01-06 21:23
categories: "2024"
tags:
  - shell
---

a simple shell version check utility function, it can be extended to suit the need:

usecase 1:
```text
#!/bin/sh

version_dot_separator() {
    echo "$@" | awk -F. '{ printf("%d%03d%03d%03d\n", $1,$2,$3,$4); }';
}

test1() {
    if [ $(version_dot_separator $1) -ge $(version_dot_separator "2406.218") ]; then
        echo 'version is up to date'
    else
        echo 'version is not up to date'
    fi
}
test1 '2409.218'   # up: correct
test1 '2403.2080'  # up: wrong
```

usecase 2:
```text
#!/bin/sh

version_dot_separator_extended() {
    # 4 digit for each separate num
    echo "$@" | awk -F. '{ printf("%d%04d%04d%04d\n", $1,$2,$3,$4); }';
}

test2() {
    if [ $(version_dot_separator_extended $1) -ge $(version_dot_separator_extended "2406.218") ]; then
        echo 'version is up to date'
    else
        echo 'version is not up to date'
    fi
}
test2 '2403.2080'  # not up: correct
```

usecase 3:
```text
#!/bin/sh

version_dash_separator() {
    echo "$@" | awk -F- '{ printf("%d%03d%03d%03d\n", $1,$2,$3,$4); }';
}

test3() {
    if [ $(version_dash_separator $1) -ge $(version_dash_separator "2406.218") ]; then
        echo 'version is up to date'
    else
        echo 'version is not up to date'
    fi
}
test3 '2406-218'  # up: correct
```
