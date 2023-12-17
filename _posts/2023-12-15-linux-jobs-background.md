---
layout: post
title: "exit jobs that cannot accept SIGTERM to exit (linux)"
author: "melon"
date: 2023-12-15 20:56
categories: "2023"
tags:
  - linux
---

### # how to exit jobs that cannot use ctrl+c to interrupt
using wrong cmd combination cause ctrl+c cannot exit:
```text
$ sudo vim /etc/gitlab-runner/config.toml | tail -n 21
  Vim: Warning: Output is not to a terminal

  <the program hangs, even ctrl+c signal cannot terminate it>
```
more details about judging whether a fd of current process is connected to a terminal,
please search 'connect to terminal' in blog.

<hr>

### # solution
using ctrl+z to make the tail as background job:
```text
$ sudo vim /etc/gitlab-runner/config.toml | tail -n 21
  Vim: Warning: Output is not to a terminal
  ...
  <press ctrl+z, let it running on background>
  [1]+  Stopped                 sudo vim /etc/gitlab-runner/config.toml | tail -n 21
```

thus we could check job pid, the outer tail process is already terminated by SIGTERM:
```text
$ jobs -l
  [1]+ 11059 Stopped (tty output)    sudo vim /etc/gitlab-runner/config.toml
       11060 Terminated              | tail -n 21

$ kill %1          # use kill to terminate processes
```
