---
layout: post
title: "file descriptor info (linux, fd)"
author: "melon"
date: 2024-01-05 18:37
categories: "2024"
tags:
  - linux
---

in linux, file descriptor is a process-unique identifier (handle) for a file or other input/output resources.
for more info about fd definitions, please ref blog post: linux-file-descriptor.
this article mainly focus on the script & tool & cmd for fd info examination.

<hr>

### # examine file descriptor by procfs
1 check all open file info of current shell process, the process is connected to a pseudo-terminal:

```text
$ ls -l /proc/$$/fd/
lrwx------ 1 melon melon 64 Jul  9 21:15 0 -> /dev/pts/0
lrwx------ 1 melon melon 64 Jul  9 21:15 1 -> /dev/pts/0
lrwx------ 1 melon melon 64 Jul  9 21:15 2 -> /dev/pts/0
```

<p style="margin-bottom: 20px;"></p>

2 check certain fd info of current shell process:

```text
$ cat /proc/$$/fdinfo/0
pos:	0
flags:  02
mnt_id: 28
```

the flags above only contains status flags, which make it hard for get some details.
the next section provide a python script to decode the flags.

<hr>

### # exmaine file descriptor flags by python decoder script
1 open some files with flags, return the pid & fd for latter usage:

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

<p style="margin-bottom: 20px;"></p>

2 decode the flags of the returned fd of a process in human-readable format:

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

<p style="margin-bottom: 20px;"></p>

3 test the functionality of the above function:

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

thus, by the decoder above, it's easy to determine whether a socket fd is working on blocking mode:

```text
$ sudo python3 ./flags.py 943 6
flags mask: 02004002
os.O_RDWR is set
os.O_NDELAY is set
os.O_NONBLOCK is set
os.O_CLOEXEC is set
```

<hr>

### # examine the underlying resource pointed by certain file descriptor
1 whether a fd point to a terminal device?  
a) it's easy to check whether a fd is associated with tty device by shell build-in method:

```text
fd="$1"
if [ -t "$1" ]; then
    echo 'the input fd is point to a terminal'
else
    echo 'the input fd is not point to a terminal!'
fi
```

b) to dig deeper, check the libcall utilized of the above shell script:

```text
$ ltrace [ -t 1 ] | cat
[...]
isatty(1)                                      = 0
[...]
```

c) furthermore, check the syscall underneath of the isatty() libcall:

```text
$ strace [ -t 1 ] | cat
execve("/usr/bin/[", ["[", "-t", "1", "]"], 0x7ffee89f82c8 /* 41 vars */) = 0
brk(NULL)                               = 0x1fab000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f41bd7b0000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=134291, ...}) = 0
mmap(NULL, 134291, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f41bd78f000
close(3)                                = 0
open("/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0`&\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=2156592, ...}) = 0
mmap(NULL, 3985920, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f41bd1c2000
mprotect(0x7f41bd386000, 2093056, PROT_NONE) = 0
mmap(0x7f41bd585000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1c3000) = 0x7f41bd585000
mmap(0x7f41bd58b000, 16896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f41bd58b000
close(3)                                = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f41bd78e000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f41bd78c000
arch_prctl(ARCH_SET_FS, 0x7f41bd78c740) = 0
access("/etc/sysconfig/strcasecmp-nonascii", F_OK) = -1 ENOENT (No such file or directory)
access("/etc/sysconfig/strcasecmp-nonascii", F_OK) = -1 ENOENT (No such file or directory)
mprotect(0x7f41bd585000, 16384, PROT_READ) = 0
mprotect(0x608000, 4096, PROT_READ)     = 0
mprotect(0x7f41bd7b1000, 4096, PROT_READ) = 0
munmap(0x7f41bd78f000, 134291)          = 0
brk(NULL)                               = 0x1fab000
brk(0x1fcc000)                          = 0x1fcc000
brk(NULL)                               = 0x1fcc000
open("/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=106176928, ...}) = 0
mmap(NULL, 106176928, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f41b6c7f000
close(3)                                = 0
ioctl(1, TCGETS, 0x7fff4ce6a8b0)        = -1 ENOTTY (Inappropriate ioctl for device)
close(1)                                = 0
close(2)                                = 0
exit_group(1)                           = ?
+++ exited with 1 +++
```

hence, the syscall ioctl is invoked by libcall isatty to check whether current fd is linked to tty.

<p style="margin-bottom: 20px;"></p>

2 judging whether a fd point to a pipe buffer  
the fstat syscall can be used to determine whether a fd is associated with a pipe or fifo.
it will returns a struct containing a field st_mode:

```text
struct stat {
    dev_t     st_dev;                     // ID of device containing file
    ino_t     st_ino;                     // inode number
    mode_t    st_mode;                    // fd type, file permissions related to this fd
    nlink_t   st_nlink;                   // number of hard links
    uid_t     st_uid;                     // user ID of owner
    gid_t     st_gid;                     // group ID of owner
    dev_t     st_rdev;                    // device ID (if special file)
    off_t     st_size;                    // total size, in bytes
    blksize_t st_blksize;                 // blocksize for file system I/O
    blkcnt_t  st_blocks;                  // number of 512B blocks allocated
    time_t    st_atime;                   // time of last access
    time_t    st_mtime;                   // time of last modification
    time_t    st_ctime;                   // time of last status change
};
```

using info returned by fstat syscall, the S_ISFIFO() standard c macro can be used on the st_mode field
to determine if the fd is a pipe/fifo.

a) show available fd of current shell process:

```text
$ ls -l /proc/self/fd
total 0
lrwx------ 1 metung metung 64 Jul 10 17:07 0 -> /dev/pts/25
lrwx------ 1 metung metung 64 Jul 10 17:07 1 -> /dev/pts/25
lrwx------ 1 metung metung 64 Jul 10 17:07 2 -> /dev/pts/25
lr-x------ 1 metung metung 64 Jul 10 17:07 3 -> /proc/5375/fd
```

b) examine the file property by gnu stat:

```text
$ stat -c %F - <& 0
character special file

$ stat -c %F - <& 1
character special file

$ stat -c %F - <& 2
character special file
```

<p style="margin-bottom: 20px;"></p>

3 whether a fd is seekable?  
pipe, socket, tty device are not seekable, regular file and most block device generally are seekable.

an innocuous way to judge seekable or not: issue a relative syscall lseek with 0 offset over the fd.

```text
off_t lseek(int fd, off_t pos, int whence)   // off_t: offset type for storing file positions
```

if lseek return -1, then the fd related resource is not seekable or the position for seek is out of range
for the resource.

then it's easy to think of using gnu dd (a standard utility to invoke syscall lseek), however,
it wont work at all: according to gnu dd src, it wont call lseek when asking for an offset of 0.
