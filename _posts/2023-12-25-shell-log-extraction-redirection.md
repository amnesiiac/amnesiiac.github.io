---
layout: post
title: "shell log extraction & redirection (shell)"
author: "melon"
date: 2023-12-25 21:40
categories: "2023"
tags:
  - shell
---

### # use awk for stdin & log separated direction
seq output 1 to 4 to stdout:
```text
$ seq 4
1
2
3
4
```

awk to print the line matching pattern 3 to stdin, and redirect wholesome (matched, not matched) to log (new):
```text
$ seq 4 2>&1 | awk '/3/ {print;} {print > "log"}'
3

# file log contains wholesome output from seq 4
$ cat log
1
2
3
4
```

awk to print the line matching pattern 3 to stdin, and redirect part (only no matched) to log (new):
```text
$ seq 4 2>&1 | awk '/3/ {print; next} {print > "log"}'
3

# file log contains only part of output from seq 10
$ cat log
1
2
4
```

<hr>

### # explanation about shell next
ref: https://www.tecmint.com/use-next-command-with-awk-in-linux/

<hr>

### # use shell variables inside awk, generate log to both stdout & logfile
```text
$ logfile=/home/metung/log

$ echo $logfile
/home/metung/log

$ seq 10 2>&1 | awk -v log="$logfile" '/3/ {print; next} {print > "log"}'
awk: fatal: cannot use gawk builtin `log' as variable name

$ seq 10 2>&1 | awk -v mlog="$logfile" '/3/ {print; next} {print > "mlog"}'
3

$ seq 10 2>&1 | awk -v mlog="$logfile" '/3|4|5/ {print; next} {print > "mlog"}'
3
4
5
```

<hr>

### # use tail -f background for stdout, use cmd redirection for logs
```text
$ touch test.log

# listening on log files, pipe to grep, then direct to stdout (running background)
$ tail -f test.log | grep -e '3' -e '2' &

# mimic program output, redirect to file, the output is written by tail
$ seq 10 >> test.log 2>&1
3
2
```

<hr>

### # code in action
see personal proj: rebornlinux/aports/.gitlab-ci.yml -> .init stage for details.
