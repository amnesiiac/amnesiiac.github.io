---
layout: post
title: "tail -f implementation for log text examination (python)"
author: "melon"
date: 2025-05-21 17:05
categories: "2025"
tags:
  - python
---

a python toy code to mimic shell implementation of tail -f `${file}`:

<hr>

### # 1: a simple tail -f over a flowing log file

```text
while 1:
    where = file.tell()        # store current position in file
    line = file.readline()     # try to read a line
    if not line:               # if no new line is found
        time.sleep(1)          # wait 1 second
        file.seek(where)       # go back to previous position
    else:
        print(line, end='')    # print line neatly
```

<hr>

### # 2: yield in while loop as generator of each line of flowing log file

```text
def tail_f(file):
    interval = 1.0

    while True:
        where = file.tell()
        line = file.readline()
        if not line:
            time.sleep(interval)
            file.seek(where)
        else:
            yield line

for line in tail_f(open(sys.argv[1])):
    print(line, end='')
```

<hr>

### # 3: tail -10 filename to enable backtrace of 10 lines scrolling display of flowing log

```text
import sys
def tail_lines(filename, linesback=10, returnlist=0):
    """
    tail -10 filename
    filename   file to read
    linesback  num of lines to read from end of file
    returnlist return a list containing the lines instead of a string
    """
    avgcharsperline=75

    with open(filename, 'r') as file:
        while 1:
            try:
                file.seek(-1 * avgcharsperline * linesback, 2)
            except IOError:
                file.seek(0)
            if file.tell() == 0:
                atstart=1
            else:
                atstart=0
            lines = file.read().split("\n")
            if(len(lines) > (linesback+1)) or atstart:
                break
            avgcharsperline = avgcharsperline * 1.3    # inc avg line len for retry
    file.close()

    if len(lines) > linesback:
        start = len(lines) - linesback - 1
    else:
        start = 0
    if returnlist:
        return lines[start:len(lines)-1]

    out = ""
    for l in lines[start:len(lines)-1]:
        out=out + l + "\n"
    return out

print(tail_lines('/etc/hosts',5,1))
sys.stdout.write(tail_lines('/etc/hosts',5))
```
