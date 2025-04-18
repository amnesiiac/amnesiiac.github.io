---
layout: post
title: "function flow control determine by invoke times (python, trick)"
author: "melon"
date: 2023-10-21 22:33
categories: "2023"
tags:
  - python
---

test.py:

```text
def hello_or_fuck():
    if not hasattr(hello_or_fuck, 'called'):         # set func.attr invoke times
        hello_or_fuck.called = 0
    hello_or_fuck.called += 1                        # update func attr
    if hello_or_fuck.called == 1:                    # flow control by invoke times
        print("hello")
    else:
        print("fuck")

hello_or_fuck()                                      # invoke 3 times
hello_or_fuck()
hello_or_fuck()
```

output:

```text
$ python3 test.py
hello
fuck
fuck
```
