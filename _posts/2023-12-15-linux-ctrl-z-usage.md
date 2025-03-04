---
layout: post
title: "terminate jobs that cannot handle sigint & exit (linux, signal)"
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
using ctrl+z to make the tail suspended (sending SIGTSTP):

```text
$ sudo vim /etc/gitlab-runner/config.toml | tail -n 21
  Vim: Warning: Output is not to a terminal
  ...
  <press ctrl+z, let it stop at background>
  [1]+  Stopped                 sudo vim /etc/gitlab-runner/config.toml | tail -n 21
```

thus we could check job pid, the outer tail process is already terminated by SIGTERM:

```text
$ jobs -l
  [1]+ 11059 Stopped (tty output)    sudo vim /etc/gitlab-runner/config.toml
       11060 Terminated              | tail -n 21

$ kill %1          # use kill to terminate processes
```

<hr>

### # application of ctrl+z
when do some experiments on a file opened by vim, a common workflow might be:

```text
vim file => edit lines => close vim => do some test based on changed file => undo the changes by imagine
```

however, some times we could not remember exactly all the way back, here's a shortcut:

```text
vim file => edit lines => ctrl+z to make it background => do some test => fg %1 && undo the changes by typing 'u'
```
