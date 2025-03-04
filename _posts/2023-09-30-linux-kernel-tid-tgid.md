---
layout: post
title: "pid vs tgid (linux, process)"
author: "melon"
date: 2023-09-30 22:04
categories: "2023"
tags:
  - linux
---

a user calls a pid is not what the kernel calls a pid, the userspace pid is actually the tgid in kernel.

in the kernel, each thread has its own id, called a pid, although it would possibly make more sense
to call this a tid, or thread id.
all the kernel pid hold a tgid (thread group id), which is the pid of the first thread that was created in
current threads group.

<hr>

### # a graph illustration of different view of pid from kernel & userspace

```txt

                                        |
  view from userspace     <-- PID 43 -->|<----------------- PID 42 ----------------->  (pid = kernel tgid)
                                        |                           |
                                        |      +---------+          |
                                        |      | process |          |
                                        |     _| pid=42  |_         |
                                   __(fork) _/ | tgid=42 | \_ (new thread) _
                                  /     |      +---------+          |       \
                          +---------+   |                           |    +---------+
                          | process |   |                           |    | process |
                          | pid=43  |   |                           |    | pid=44  |
                          | tgid=43 |   |                           |    | tgid=42 |
                          +---------+   |                           |    +---------+
                                        |                           |
  view from kernel space  <-- PID 43 -->|<--------- PID 42 -------->|<--- PID 44 --->  (pid = tid)
                                        |                           |
```

<hr>

### # conclusion
1 when new process is created, both the pid and tgid of the thread are the same number.  
2 when the thread starts another thread, the new thread get its own pid so that the scheduler can schedule
it independently, but inherits the tgid from the original thread.  

in this way, the kernel can happily schedule threads independent of what process they belong to,
while processes (thread group ids) are reported to the userspace.
