---
layout: post
title: "os.listdir(dir) permission issue when dir is unprivileged (python, issue)"
author: "melon"
date: 2023-10-22 22:03
categories: "2023"
tags:
  - python
---

a python toy code to illustrate a issue when calling os.listdir(dir) on the root onwed dir:

```text
import os

os.mkdir('unreadable_dir')          # dir creation
os.chmod('unreadable_dir', 0)       # set permission as root

result = os.listdir('.')            # call os.listdir on . wont cause the any problem
print(result)                       # ["unreadable_dir", ".", "..", ...]

try:
    os.listdir('unreadable_dir')    # call os.listdir right on the root owned dir will report err
                                    # Traceback (most recent call last):
                                    #   File "<stdin>", line 1, in <module>
                                    # OSError: [Errno 13] Permission denied: 'unreadable_dir'
except Exception as e:
    print(f"error try list dir based on a non-readable dir: {e}")
```

the above script use try except to avoid runtime issues cause program panic.
