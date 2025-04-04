---
layout: post
title: "strace: tracer for syscall (linux)"
author: "melon"
date: 2023-12-28 21:21
categories: "2023"
tags:
  - linux
  - kernel
---

### # strace
strace is a userspace tracker, used to monitor the interaction between userspace proc and kernel,
e.g. syscall, signal, proc status changing...

<hr>

### # basic usages (todo: update in shuttle)
take qemu-system-x86 process as a case (sleeping):
```text
$ top
11385 root      20   0 5598792   1.6g  11400 S   6.7  1.6 860:41.00 qemu-system-x86

$ cat /proc/11385/status
Name:   qemu-system-x86
Umask:  0022
State:  S (sleeping)
Tgid:   11385
...
```

1 get statistics of all types of syscalls of a proc (timespan):
```text
$ strace -c -p 11385
strace: Process 11385 attached
strace: Process 11385 detached
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 95.11    0.069545        1121        62           ppoll
  2.89    0.002114          15       134        52 read
  1.76    0.001286          15        82           ioctl
  0.21    0.000151          37         4           writev
  0.03    0.000025          25         1           futex
------ ----------- ----------- --------- --------- ----------------
100.00    0.073121                   283        52 total
```
the proc is mainly on ppoll system call, which is commonly used in event-driven programming and asynchronous I/O
to efficiently wait for multiple fd and handle timeouts and signals in a controlled manner.

2 simple monitoring all syscall invoke details for certain proc:
```text
$ strace -p 11385
strace: Process 11385 attached
ppoll([{fd=15, events=POLLIN|POLLERR|POLLHUP}, ... {fd=47, events=POLLIN}], 37,
      {tv_sec=0, tv_nsec=421336447}, NULL, 8)=1 ([{fd=17, revents=POLLIN}], left {tv_sec=0, tv_nsec=375787801})
read(16, "\0\0\0\0\0\0\0\0\0\0\6\0\0\0\0\20027\322\26<\221\10\0E\0\0,\304\301@\0"..., 69632) = 68
ioctl(13, KVM_SIGNAL_MSI, 0x7ffc4d030810) = 1
...
```

3 trace current proc & follow child proc (spawned by fork, vfork, clone syscall):
```text
$ strace -f -p 11385
strace: Process 11385 attached with 3 threads
[pid 12325] ioctl(23, KVM_RUN <unfinished ...>
[pid 12170] futex(0x5630b77805c8, FUTEX_WAIT, 4294967295, NULL <unfinished ...>
[pid 11385] ppoll([{fd=15, events=POLLIN|POLLERR|POLLHUP}, ... {fd=47, events=POLLIN}], 37,
           {tv_sec=0, tv_nsec=421336447}, NULL, 8)=1 ([{fd=17, revents=POLLIN}], left {tv_sec=0, tv_nsec=375787801})
[pid 12325] <... ioctl resumed>, 0)     = 0
[pid 12325] ioctl(23, KVM_RUN, 0)       = 0
...
[pid 12325] ioctl(23, KVM_RUN, 0)       = 0
[pid 12325] ioctl(23, KVM_RUN <unfinished ...>
[pid 11385] <... ppoll resumed>)        = 1 ([{fd=16, revents=POLLIN}], left {tv_sec=0, tv_nsec=127161615})
[pid 11385] read(16, "\0\0\0\0\0\0\0\0\0\0\6\0\0\0\0\20027\322\26<\221\10\0E\0\0,\345\306@\0"..., 69632) = 68
[pid 12325] <... ioctl resumed>, 0)     = 0
...
```

4 trace process with timestamp (s)
```text
$ strace -t -p 11385
strace: Process 11385 attached
18:39:54 ppoll([{fd=15, events=POLLIN|POLLERR|POLLHUP}, ... {fd=47, events=POLLIN}], 37,
        {tv_sec=0, tv_nsec=421336447}, NULL, 8)=1 ([{fd=17, revents=POLLIN}], left {tv_sec=0, tv_nsec=375787801})
18:39:54 read(16, "\0\0\0\0\0\0\0\0\0\0\6\0\0\0\0\200^\266\1+Qf\10\0E\0\0,\3428@\0"..., 69632) = 68
18:39:54 ioctl(13, KVM_SIGNAL_MSI, 0x7ffc4d030810) = 1
...
```

5 trace process with timestamp (us)
```text
$ strace -tt -p 11385
strace: Process 11385 attached
18:43:27.017370 ppoll([{fd=15, events=POLLIN|POLLERR|POLLHUP}, ... {fd=47, events=POLLIN}], 37,
               {tv_sec=0, tv_nsec=421336447}, NULL, 8)=1 ([{fd=17, revents=POLLIN}], left {tv_sec=0, tv_nsec=375787801})
18:43:27.172187 read(16, "\0\0\0\0\0\0\0\0\0\0\6\0\0\0\0\200^\266\1+Qf\10\0E\0\0<Y\350@\0"..., 69632) = 84
...
```

6 trace process with timestamp (relative to system bootup time, us)
```text
$ strace -ttt -p 11385
strace: Process 11385 attached
1703760568.552415 ppoll([{fd=15, events=POLLIN|POLLERR|POLLHUP}, ... {fd=47, events=POLLIN}], 37,
           {tv_sec=0, tv_nsec=421336447}, NULL, 8)=1 ([{fd=17, revents=POLLIN}], left {tv_sec=0, tv_nsec=375787801})
1703760568.604994 read(17, "\0\0\0\0\0\0\377\306\312_\367Y\335\201\0\0\220\210\0\0\0\0\0"..., 69632) = 102
1703760568.605103 ioctl(13, KVM_SIGNAL_MSI, 0x7ffc4d030810) = 1
...
```

7 trace process with time interval for each syscall <time>
```text
$ strace -T -p 11385
strace: Process 11385 attached
ppoll([{fd=15, events=POLLIN|POLLERR|POLLHUP}, ... {fd=47, events=POLLIN}], 37,
   {tv_sec=0, tv_nsec=421336447}, NULL, 8)=1 ([{fd=17, revents=POLLIN}], left {tv_sec=0, tv_nsec=375787801}) <0.096842>
read(16, "\0\0\0\0\0\0\0\0\0\0\6\0\0\0\0\200^\266\1+Qf\10\0E\0\0<\223O@\0"..., 69632) = 84 <0.000050>
ioctl(13, KVM_SIGNAL_MSI, 0x7ffc4d030810) = 1 <0.000056>
...
```

8 trace process with verbose info
```text
$ strace -v -p 11385
strace: Process 11385 attached
ppoll([{fd=15, events=POLLIN|POLLERR|POLLHUP}, ... {fd=47, events=POLLIN}], 37,
      {tv_sec=0, tv_nsec=421336447}, NULL, 8)=1 ([{fd=17, revents=POLLIN}], left {tv_sec=0, tv_nsec=375787801})
read(16, "\0\0\0\0\0\0\0\0\0\0\6\0\0\0\0\200^\266\1+Qf\10\0E\0\0<~\244@\0"..., 69632) = 84
ioctl(13, KVM_SIGNAL_MSI, 0x7ffc4d030810) = 1
...
```

9 trace process with hex output
```text
$ strace -x -p 11385
strace: Process 11385 attached
ppoll([{fd=15, events=POLLIN|POLLERR|POLLHUP}, ... {fd=47, events=POLLIN}], 37,
      {tv_sec=0, tv_nsec=421336447}, NULL, 8)=1 ([{fd=17, revents=POLLIN}], left {tv_sec=0, tv_nsec=375787801})
read(16, "\x00\x00\x06\x80\x32\x37\xd2\x16\x3c\x91\x08\x00\x45\x00\x00\x2c\x9b\x4b\x40\x00"..., 69632) = 68
ioctl(13, KVM_SIGNAL_MSI, 0x7ffc4d030810) = 1
...
```

10 trace process and specify the max string len to print
```text
$ strace -s 1024 -p 11385
strace: Process 11385 attached
ppoll([{fd=15, events=POLLIN|POLLERR|POLLHUP}, ... {fd=47, events=POLLIN}], 37,
      {tv_sec=0, tv_nsec=421336447}, NULL, 8)=1 ([{fd=17, revents=POLLIN}], left {tv_sec=0, tv_nsec=375787801})
read(17, "\0\377\377\306\220\210\1\0\tPU\4c\0\0\0", 69632) = 102
ioctl(13, KVM_SIGNAL_MSI, 0x7ffc4d030810) = 1
read(17, 0x5630b8ec43dc, 69632)         = -1 EAGAIN (Resource temporarily unavailable)
...
```
compared with strace -p 11385 result, find that all bytes of read content is printed out.

11 trace process and output trace result to file
```text
$ strace -o tracelog -p 11385
strace: Process 11385 attached
strace: Process 11385 detached

$ cat tracelog
ppoll([{fd=15, events=POLLIN|POLLERR|POLLHUP}, ... {fd=47, events=POLLIN}], 37,
      {tv_sec=0, tv_nsec=421336447}, NULL, 8)=1 ([{fd=17, revents=POLLIN}], left {tv_sec=0, tv_nsec=375787801})
read(16, "\0\0\0\0\0\0\0\0\0\0\6\0\0\0\0\200^\266\1+Qf\10\0E\0\0< \267@\0"..., 69632) = 84
ioctl(13, KVM_SIGNAL_MSI, 0x7ffc4d030810) = 1
...
```

12 trace process with fd/socket info parsed translating to file path or addr
```text
$ strace -yy -p 11385
strace: Process 11385 attached
ppoll([{fd=15<TCP:[2455742433]>, events=POLLIN|POLLERR|POLLHUP}, ...], 37,
      {tv_sec=0, tv_nsec=166212998}, NULL, 8)=1 ([{fd=16, revents=POLLIN}], left {tv_sec=0, tv_nsec=143425361})
read(16</dev/net/tun<char 10:200>>, "\0\0\0\0\0\0\0\0\0\0\6\0\0\0\0\200^\266\1+Qf\10\0E\270\0L\n7@\0"..., 69632) = 100
ioctl(13<anon_inode:kvm-vm>, KVM_SIGNAL_MSI, 0x7ffc4d030810) = 1
...
```

13 strace to startup process & follow certain list of syscall
```text
$ strace -e trace=open top >/dev/null
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
open("/lib64/libprocps.so.4", O_RDONLY|O_CLOEXEC) = 3
open("/lib64/libsystemd.so.0", O_RDONLY|O_CLOEXEC) = 3
open("/lib64/libncurses.so.5", O_RDONLY|O_CLOEXEC) = 3
open("/lib64/libtinfo.so.5", O_RDONLY|O_CLOEXEC) = 3
...
open("/proc/filesystems", O_RDONLY)     = 3
open("/proc/self/auxv", O_RDONLY)       = 3
open("/sys/devices/system/cpu/online", O_RDONLY|O_CLOEXEC) = 3
open("/proc/self/auxv", O_RDONLY)       = 3
open("/proc/self/stat", O_RDONLY)       = 3
...
open("/proc/self/status", O_RDONLY)     = 3
open("/sys/devices/system/node/node0/meminfo", O_RDONLY) = 4
open("/proc/self/status", O_RDONLY)     = 3
open("/etc/toprc", O_RDONLY)            = -1 ENOENT (No such file or directory)
open("/home/metung/.toprc", O_RDONLY)   = -1 ENOENT (No such file or directory)
open("/usr/share/terminfo/x/xterm", O_RDONLY) = 3
open("/proc/sys/kernel/pid_max", O_RDONLY) = 3
open("/dev/null", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 4
open("/proc/stat", O_RDONLY)            = 4
open("/proc/uptime", O_RDONLY)          = 5
open("/proc/1/stat", O_RDONLY)          = 7
open("/proc/1/statm", O_RDONLY)         = 7
open("/etc/nsswitch.conf", O_RDONLY|O_CLOEXEC) = 7
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 7
open("/lib64/libnss_files.so.2", O_RDONLY|O_CLOEXEC) = 7
open("/etc/passwd", O_RDONLY|O_CLOEXEC) = 7
open("/proc/2/stat", O_RDONLY)          = 7               # process status
open("/proc/2/statm", O_RDONLY)         = 7               # process mem statistics
open("/proc/3/stat", O_RDONLY)          = 7
...
```

14 strace to startup process and follow syscall related to file operations
```text
$ strace -e trace=file top > /dev/null
execve("/usr/bin/top", ["top"], 0x7ffddceae050 /* 44 vars */) = 0
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
open("/lib64/libprocps.so.4", O_RDONLY|O_CLOEXEC) = 3
open("/lib64/libsystemd.so.0", O_RDONLY|O_CLOEXEC) = 3
open("/lib64/libncurses.so.5", O_RDONLY|O_CLOEXEC) = 3
...
access("/etc/selinux/config", F_OK)     = 0
open("/proc/self/auxv", O_RDONLY)       = 3
open("/sys/devices/system/cpu/online", O_RDONLY|O_CLOEXEC) = 3
open("/proc/self/auxv", O_RDONLY)       = 3
open("/proc/self/stat", O_RDONLY)       = 3
...
open("/proc/self/status", O_RDONLY)     = 3
...
open("/proc/1/stat", O_RDONLY)          = 7
open("/proc/1/statm", O_RDONLY)         = 7
open("/etc/nsswitch.conf", O_RDONLY|O_CLOEXEC) = 7
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 7
open("/lib64/libnss_files.so.2", O_RDONLY|O_CLOEXEC) = 7
access("/etc/sysconfig/strcasecmp-nonascii", F_OK) = -1 ENOENT (No such file or directory)
open("/etc/passwd", O_RDONLY|O_CLOEXEC) = 7
stat("/proc/2", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
open("/proc/2/stat", O_RDONLY)          = 7
open("/proc/2/statm", O_RDONLY)         = 7
...
```

15 strace to starup process & follow syscall related to process operations
```text
$ make pipetest app
$ strace -e trace=process ./out
execve("./out", ["./out"], 0x7ffd8f6026c0 /* 45 vars */) = 0
arch_prctl(ARCH_SET_FS, 0x7f59bc7de740) = 0
[parent process] reading from descriptor 3 character A
forking a new child process ->
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f59bc7dea10) = 17172
[child process] reading from descriptor 3 character C
[parent process] reading from descriptor 3 character B
[child process] reading from descriptor 3 character D
[parent process] reading from descriptor 3 character E
[child process] reading from descriptor 3 character F
wait4(17172, NULL, 0, NULL)             = 17172
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=17172, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
exit_group(0)                           = ?
+++ exited with 0 +++
...
```

16 strace to startup process & follow syscall related to network operations
```text
$ strace -e trace=network ping 1.1.1.1
socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP) = -1 EACCES (Permission denied)
socket(AF_INET, SOCK_RAW, IPPROTO_ICMP) = 3
socket(AF_INET, SOCK_DGRAM, IPPROTO_IP) = 4
connect(4, {sa_family=AF_INET, sin_port=htons(1025), sin_addr=inet_addr("1.1.1.1")}, 16) = 0
getsockname(4, {sa_family=AF_INET, sin_port=htons(33388), sin_addr=inet_addr("192.168.0.20")}, [16]) = 0
setsockopt(3, SOL_RAW, ICMP_FILTER, ~(1<<ICMP_ECHOREPLY|1<<ICMP_DEST_UNREACH|1<<ICMP_SOURCE_QUENCH|1<<ICMP_REDIRECT...
setsockopt(3, SOL_IP, IP_RECVERR, [1], 4) = 0
setsockopt(3, SOL_SOCKET, SO_SNDBUF, [324], 4) = 0
setsockopt(3, SOL_SOCKET, SO_RCVBUF, [65536], 4) = 0
getsockopt(3, SOL_SOCKET, SO_RCVBUF, [131072], [4]) = 0
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
setsockopt(3, SOL_SOCKET, SO_TIMESTAMP, [1], 4) = 0
setsockopt(3, SOL_SOCKET, SO_SNDTIMEO, "\1\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", 16) = 0
setsockopt(3, SOL_SOCKET, SO_RCVTIMEO, "\1\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", 16) = 0
sendto(3, "\10\0006j7\305\0\1Qo\215e\0\0\0\0\336'\16\0\0\0\0\0\20\21\22\23\24\25\26\27"..., 64, 0, {sa_family=AF_INET...
recvmsg(3, {msg_namelen=128}, 0)        = -1 EAGAIN (Resource temporarily unavailable)
sendto(3, "\10\0~>7\305\0\2Ro\215e\0\0\0\0\225R\16\0\0\0\0\0\20\21\22\23\24\25\26\27"..., 64, 0, {sa_family=AF_INET...
recvmsg(3, ^C
strace: Process 14277 detached
--- 1.1.1.1 ping statistics ---
 <detached ...>
2 packets transmitted, 0 received, 100% packet loss, time 1011ms
...
```

17 stace to startup process & follow syscall related to signal operations
```text
$ strace -e trace=signal top > /dev/null
rt_sigaction(SIGRTMIN, {sa_handler=0x7f5d1e2f1860, sa_mask=[], sa_flags=SA_RESTORER|SA_SIGINFO, sa_restorer=...
rt_sigaction(SIGRT_1, {sa_handler=0x7f5d1e2f18f0, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART|SA_SIGINFO...
rt_sigprocmask(SIG_UNBLOCK, [RTMIN RT_1], NULL, 8) = 0
rt_sigaction(SIGRT_32, {sa_handler=0x405460, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f5d1fdaf400}...
rt_sigaction(SIGRT_31, {sa_handler=0x405460, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f5d1fdaf400}...
strace: Process 22281 detached
...
```
