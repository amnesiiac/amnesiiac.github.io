---
layout: post
title: "make: unable to compile with make, no space left on device (make)"
author: "melon"
date: 2023-12-04 20:12
categories: "2023"
tags:
  - make
---

### # background
when try compiling shuttle app, an error occured:
```text
$(shuttle) make
g++ -g -Wall -std=c++11 -pthread -w -c main.cpp -o main.o
main.cpp:319:1: fatal error: error writing to /tmp/pccmW5v18.s: No space left on device
}
^
compilation terminated.
make: *** [main.o] Error 1
```

<hr>

### # using df tools to locate which mount point is overload:
check the system disk usage by:
```text
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs         52G     0   52G   0% /dev                   <--- RAM as filesys
tmpfs            52G     0   52G   0% /dev/shm
tmpfs            52G  4.1G   48G   8% /run
tmpfs            52G     0   52G   0% /sys/fs/cgroup
/dev/vda1       744G  730G     0 100% /                      <--- system-wide /tmp is mounted on /, has no space left
/dev/loop0       56M   56M     0 100% /var/lib/snapd/snap/core18/2074
/dev/loop2       66M   66M     0 100% /var/lib/snapd/snap/gtk-common-themes/1515
/dev/loop1       56M   56M     0 100% /var/lib/snapd/snap/teams-for-linux/156
/dev/loop3      165M  165M     0 100% /var/lib/snapd/snap/gnome-3-28-1804/161
/dev/loop4       71M   71M     0 100% /var/lib/snapd/snap/teams-for-linux/172
/dev/vdb1       2.0T  1.6T  302G  85% /repo1                 <--- this dir has space left, /repo1/metung/
/dev/loop5       56M   56M     0 100% /var/lib/snapd/snap/core18/2066
/dev/loop6       40M   40M     0 100% /var/lib/snapd/snap/bcc/149
/dev/loop7      163M  163M     0 100% /var/lib/snapd/snap/gnome-3-28-1804/145
/dev/loop8       33M   33M     0 100% /var/lib/snapd/snap/snapd/12159
/dev/loop9       33M   33M     0 100% /var/lib/snapd/snap/snapd/12398
/dev/loop10      63M   63M     0 100% /var/lib/snapd/snap/gtk-common-themes/1506
tmpfs            11G   12K   11G   1% /run/user/42
tmpfs            11G     0   11G   0% /run/user/1013
tmpfs            11G   48K   11G   1% /run/user/1012
tmpfs            11G     0   11G   0% /run/user/1011
tmpfs            11G   44K   11G   1% /run/user/1003
tmpfs            11G     0   11G   0% /run/user/1019
```

<hr>

### # use findmnt tool check where does /tmp mounted on
```text
$ findmnt --target /tmp
TARGET SOURCE    FSTYPE OPTIONS
/      /dev/vda1 ext4   rw,relatime
```
that is, the /tmp dir is actually mounted under /.

<hr>

### # solution
for a temporary cure, modify ENV variable for make to work:
```text
$ mkdir /repo1/metung/tmp
$ export TMPDIR=/repo1/metung/tmp
```
try make again, the make works:
```text
$ make
g++ -g -Wall -std=c++11 -pthread -w -c main.cpp -o main.o
g++ -g -Wall -std=c++11 -pthread -w -o out main.o -lncurses -lmenu -lform
```
