---
layout: post
title: "userspace proc callstack (linux)"
author: "melon"
date: 2023-12-25 21:23
categories: "2023"
tags:
  - linux
  - debug
---

### # syscall stack trace
take my nvim container runtime as a case to check live proc syscall info:
```text
$ ps -ef | grep nvim
metung    7445     1  0 Dec18 ?      00:01:54 docker run -it --rm -v /repo1/metung/hostfw:/root/mount/ nvim:t nvim .
root      8006  7966  0 Dec18 pts/0  00:00:00 /bin/bash /scripts/entrypoint.sh nvim .
root      8096  8006  0 Dec18 pts/0  00:00:08 nvim .
root      8097  8096  0 Dec18 ?      00:18:31 nvim --embed .
root     29000 16915  0 22:56 pts/1  00:00:00 grep nvim
metung   36146     1  0 Nov15 ?      00:11:57 docker run -it --rm -v /repo1/metung/libhv:/root/mount/ nvim:t nvim .
root     36212 36190  0 Nov15 pts/0  00:00:00 /bin/bash /scripts/entrypoint.sh nvim .
root     36296 36212  0 Nov15 pts/0  00:00:44 nvim .
root     36305 36296  0 Nov15 ?      03:22:39 nvim --embed .
```

strace to listen for nvim proc, if no operations on nvim, nothing print out:
```text
$ strace -v -p 8096
strace: Process 8096 attached
epoll_pwait(3,
```

after some r/w operation on nvim, something new on strace happened:
```text
$ strace -v -p 8096
strace: Process 8096 attached
epoll_pwait(3, [{EPOLLIN, {u32=0, u64=0}}], 1024, -1, NULL, 8) = 1
read(0, "\10", 4095)                    = 1
write(10, "\223\2\252nvim_input\221\245<C-h>", 20) = 20
epoll_pwait(3, [{EPOLLIN, {u32=12, u64=12}}], 1024, -1, NULL, 8) = 1
epoll_pwait(11, [], 1024, 0, NULL, 8)   = 0
...
epoll_pwait(3, [{EPOLLIN, {u32=12, u64=12}}], 1024, -1, NULL, 8) = 1
read(12, "\223\2\246redraw\334\0\2\334\0\r\251grid_line\224\1\21\20\334\0\25"..., 65536) = 1002
write(16, "\33[18;17Hdefault_values.yaml \342\224\200\33"..., 629) = 629
epoll_pwait(11, [], 1024, 0, NULL, 8)   = 0
write(16, "\33[34;135H\33[17X1,11          All\33"..., 38) = 38
epoll_pwait(11, [], 1024, 0, NULL, 8)   = 0
epoll_pwait(3,
```

cautious point when checking ongoing syscall callstack for certain proc:  
1 whether a syscall is ongoing.  
2 if a certain syscall is repetitively invoked (wont stop), most likely the app is not getting stuck, but recognized as busy.  
3 if the desired syscall is not seen, or the syscall is seen but no return, or do stuffs that busy consuming cpu:
e.g. compute, no IO operations), need to dive into deeper into further callstack.

<hr>

### # userspace callstack: tools to be concluded
```text
$ callstack_svr 32587  # todo
Callstack:
    ip[00]: 0xf32e3cc  (sp=0xffffe2d0)  __libc_freeres
    ip[01]: 0xf66de90  (sp=0xffffe300)  udrv_file_read+0xb8
    ip[02]: 0xf67ad4c  (sp=0xffffe340)  sysfs_led_find_led_state+0x1e94
    ip[03]: 0xf67aea4  (sp=0xffffe3d0)  sysfs_led_find_led_state+0x1fec
    ip[04]: 0xf6b7ef8  (sp=0xffffe400)  udrv_ll_dev_read+0xa8
    ip[05]: 0xf6861b0  (sp=0xffffe460)  sysfs_led_find_led_state+0xd2f8
    ...
    ip[13]: 0xfae0ee4  (sp=0xffffe6e0)  ytime_register_timezone_cb+0x680
    ip[14]: 0xf9f2fc8  (sp=0xffffe740)  event_reinit+0x734
    ip[15]: 0xf9f3774  (sp=0xffffe7a0)  event_base_loop+0x60c
    ip[16]: 0xfad32f4  (sp=0xffffe800)  event_process_loop+0x90
    ip[17]: 0xfad3428  (sp=0xffffe830)  yloop_main+0xd4
    ip[18]: 0xf19d330  (sp=0xffffe850)  __libc_init_first+0x370
    ip[19]: 0xf19d4e8  (sp=0xffffea80)  __libc_start_main+0xb8
    ip[20]: (nil)  (sp=0xffffeaa0)
```

<hr>

### # kernel space callstack:
check kernel stack info using proc sysfs:
```text
$ cat /proc/8096/stack
[<0>] ep_poll+0x425/0x470
[<0>] do_epoll_wait+0xb8/0xd0
[<0>] __x64_sys_epoll_pwait+0x4c/0xa0
[<0>] do_syscall_64+0x60/0x1b0
[<0>] entry_SYSCALL_64_after_hwframe+0x44/0xa9
```
