---
layout: post
title: "filelock wrapper implementation (python, filelock)"
author: "melon"
date: 2023-10-07 21:27
categories: "2023"
tags:
  - python
  - sync
---

a filelock wrapper implmentation in python, used to protect the critical section operations among
all processes on the same host machine.
the filelock is deisnged as a wrapper class around python module fcntl, originate from c api.

<hr>

### # filelock code in action
1 a toy filelock implementation using lower-level fcntl module:

```text
#!/usr/bin/env python3
import os, stat
import fcntl
import multiprocessing
import time
import random
import string

class FileLock:
    def __init__(self, filename):
        self.filename = filename
        self.locked = False
        if not os.path.exists(filename):                         # create lock file if not existed
            file = open(filename, "w")
            file.write("0")
            file.close()
            os.chmod(self.filename, stat.S_IRWXU | stat.S_IRWXG | stat.S_IRWXO)
    
    def lock(self):
        if self.locked:                                          # avoid re-acquire the lock when holding it
            return
        self.file = open(self.filename, "w")
        fcntl.flock(self.file.fileno(), fcntl.LOCK_EX)           # lock the process-local fd exclusively
        self.locked = True
        return self
    
    def unlock(self):
        if self.locked:                                          # not unlock over a unlocked state
            fcntl.flock(self.file.fileno(), fcntl.LOCK_UN)
            self.file.close()
            self.locked = False
    
    def __enter__(self):                                         # enter with statement
        self.lock()
    
    def __exit__(self, exc_type, exc_value, traceback):          # exit with statement
        self.unlock()
        return None

def get_int(len=5, fro=None, to=None):                           # return random int
    if fro == None or to == None:
        ret = "".join(random.SystemRandom().choice(string.digits) for _ in range(len))
    else:
        ret = random.randint(fro, to)
    return int(ret)

def pfunc(flk, n, switch):
    times = 0
    while times < 3:
        if switch:
            with flk:
                d = get_int(3) / 1000
                print("+[{}]".format(n))
                time.sleep(d)
                print("-[{}]".format(n))
                time.sleep(d)
                times += 1
        else:
            d = get_int(3) / 1000
            print("+[{}]".format(n))
            time.sleep(d)
            print("-[{}]".format(n))
            time.sleep(d)
            times += 1

if __name__ == "__main__":
    lockswitch = False                                           # enable filelock or not
    flk = FileLock("/tmp/test_filelock")                         # init filelock, with backup file
    
    p1 = multiprocessing.Process(target=pfunc, args=[flk, 1, lockswitch])
    p2 = multiprocessing.Process(target=pfunc, args=[flk, 2, lockswitch])
    p3 = multiprocessing.Process(target=pfunc, args=[flk, 3, lockswitch])
    
    p1.start()                                                   # start
    p2.start()
    p3.start()
    
    time.sleep(12)
    
    p1.terminate()                                               # termintate
    p2.terminate()
    p3.terminate()
```

<p style="margin-bottom: 20px;"></p>

2 test result  
without filelock protection (setting lockswitch = False), the global write protection is failed:

```text
$(terminal#1) python3 fl.py
+[1]
+[2]
+[3]
-[1]
+[1]
-[1]
+[1]
-[3]
-[2]
-[1]
+[3]
+[2]
-[2]
+[2]
-[3]
-[2]
+[3]
-[3]

$(terminal#2) python3 fl.py
+[1]
+[2]
+[3]
-[3]
-[2]
+[3]
-[1]
+[2]
-[2]
-[3]
+[2]
+[3]
+[1]
-[1]
-[2]
+[1]
-[3]
-[1]
```

with filelock protection, the global lock contention among processes is fixed:

```text
$(terminal#1) python3 fl.py
+[1]
-[1]
+[2]
-[2]
+[3]
-[3]
+[1]
-[1]
+[2]
-[2]
+[3]
-[3]
+[1]
-[1]
+[2]
-[2]
+[3]
-[3]

$(terminal#2) python3 fl.py
+[1]
-[1]
+[2]
-[2]
+[3]
-[3]
+[1]
-[1]
+[2]
-[2]
+[3]
-[3]
+[1]
-[1]
+[2]
-[2]
+[3]
-[3]
```

<hr>

### # fcntl in kernel
ref: https://github.com/torvalds/linux/blob/master/fs/fcntl.c

<hr>

### # filelock in kernel
ref: https://github.com/torvalds/linux/blob/master/include/linux/filelock.h
