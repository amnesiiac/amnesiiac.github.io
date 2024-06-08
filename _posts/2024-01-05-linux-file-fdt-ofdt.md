---
layout: post
title: "fd & fdt & ofdt (linux, file)"
author: "melon"
date: 2024-01-05 18:37
categories: "2024"
tags:
  - linux
  - ongoing
---

file descriptor: a positive integer number used by file system calls.  
file descriptor table: each process has its own fdt with fd as its entry.  
open file descriptor table: 
inode: 

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/fdt.pdf" width="750"/>

(1) the first 3 fd stdin(0), stdout(1) and stderr(2) are special fd.
in this case, all of them point to a pseudo-terminal (/dev/pts/0).
they have no positions in open fd table due to their character device type.
thus, proc#l and proc#2 must be running under the terminal sessions.

(2) normally, only open fd table entries got their own flag, however, some fd can
have per-process flags. till now there is only one such flag: close-on-exec (O_CLOEXEC).
in this case, the fd 9 of process 1 and fd 3 of process 2 are point to the same
system-wide open fd table entries.

(3) even though the fd algorithm constantly reuses the fd and allocates them sequentially
from the lowest available, it doesn’t mean that fd number gaps cannot occur.
in this case, the fd 9 of the process 1 goes after fd 3.  
one reason for this could be: the files used fd 4, 5, 6 and 7 are already closed.
another reason for achieving this can be an explicit duplication of a fd using dup2,
dup3 or fcntl with F_DUPFD. using these syscalls, we can specify the wanted fd number.

(4) a process can have >=1 fd points to the same entry in open file descriptions.
system calls dup, dup2, dup3 and fcntl with F_DUPFD help with that.  
in this case, fd 0 and fd 2 of the process 2 refer to the same pseudo terminal entry.

(5) stdout fd might point to a regular file or pipe but not a terminal.
in this case, the stdout of the process 2 refers to a file on disk /tmp/out.txt.

(6) fd from various processes can point to the same entry in system-wide open fd table.
this can be achieved by fork syscall to inherit fd from parent proc to its child. there
are other ways to do this????

(7) in this case the file path is shown for similarity, but actually kernel uses inode,
minor and major number of a device.

(8) running in the shell context, 0,1 and 2 fd pointed to a pseudo-terminal.

(9) multiple open fd entries can be linked with the same physical file on disk.
the kernel allows us to open a file with different flags and at various offset positions.

<hr>

### # examine fd info by procfs
check all open file info of current shell process, the process is connected to a
pseudo-terminal:
```text
$ ls -l /proc/$$/fd/
lrwx------ 1 melon melon 64 Jul  9 21:15 0 -> /dev/pts/0
lrwx------ 1 melon melon 64 Jul  9 21:15 1 -> /dev/pts/0
lrwx------ 1 melon melon 64 Jul  9 21:15 2 -> /dev/pts/0
```

check certain fd info of current shell process:
```text
$ cat /proc/$$/fdinfo/0
pos:	0
flags:  02
mnt_id: 28
```
the flags above only contains status flags.

try open some files with flags, return the pid & fd:
```text
import os
import sys
import time

def open_with_status_flag(file_path):
    print(os.getpid())
    fd = os.open(file_path,  os.O_APPEND | os.O_RSYNC | os.O_NOATIME )
    with os.fdopen(fd, "r+") as f:
        print(f.fileno())
        time.sleep(9999)

if __name__ == '__main__':
    file_path = sys.argv[1]
    pid, fd = open_with_status_flag(file_path)
```

then decode the flags of certain fd of a process in human-readable format:
```text
import os
import sys

def decode_flags(pid, fd):
    with open(f"/proc/{pid}/fdinfo/{fd}", "r") as f:
        flags = f.readlines()[1].split("\t")[1].strip()
        print(f"flags mask: {flags}")
        flags = int(flags, 8)

        if flags & os.O_RDONLY:
            print("os.O_RDONLY is set")
        if flags & os.O_WRONLY:
            print("os.O_WRONLY is set")
        if flags & os.O_RDWR:
            print("os.O_RDWR is set")
        if flags & os.O_APPEND:
            print("os.O_APPEND is set")
        if flags & os.O_DSYNC:
            print("os.O_DSYNC is set")
        if flags & os.O_RSYNC:
            print("os.O_RSYNC is set")
        if flags & os.O_SYNC:
            print("os.O_SYNC is set")
        if flags & os.O_NDELAY:
            print("os.O_NDELAY is set")
        if flags & os.O_NONBLOCK:
            print("os.O_NONBLOCK is set")
        if flags & os.O_ASYNC:
            print("os.O_ASYNC is set")
        if flags & os.O_DIRECT:
            print("os.O_DIRECT is set")
        if flags & os.O_NOATIME:
            print("os.O_NOATIME is set")
        if flags & os.O_PATH:
            print("os.O_PATH is set")
        if flags & os.O_CLOEXEC:  # close on exec
            print("os.O_CLOEXEC is set")

if __name__ == '__main__':
    pid = sys.argv[1]
    fd = sys.argv[2]
    decode_flags(pid, fd)
```

usecase of the above 2 utility function:
```text
$ python3 ./open.py /tmp/tmpfile
925
3
```
```text
$ python3 ./flags.py 925 3
flags mask: 07112000
os.O_APPEND is set
os.O_DSYNC is set
os.O_RSYNC is set
os.O_SYNC is set
os.O_NOATIME is set
os.O_CLOEXEC is set
```

with the above utility program, we can determine whether a socket fd is working on blocking
mode:
```text
$ sudo python3 ./flags.py 943 6
flags mask: 02004002
os.O_RDWR is set
os.O_NDELAY is set
os.O_NONBLOCK is set
os.O_CLOEXEC is set
```

<hr>

### # todo
