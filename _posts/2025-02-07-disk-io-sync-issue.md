---
layout: post
title: "disk io cache issue (linux, disk io, sync)"
author: "melon"
date: 2025-02-07 22:15
categories: "2025"
tags:
  - linux
  - sync
---

os use cache for most recently used obj to enhance the performance, which store them into the page cache.
however, this might be confused when you see the file are already exist in devices,
but actually it still store in cache rather then nfs or smb or overlay fs.

if a script start right after the file write operation done, and to try to access or use it immediately,
then there's a chance to report file not exist error or file broken problem.

sync usage recommended scenarios:

```text
disk operation scenarios                                      | whether use sync?
---                                                           | ---
access file after untar to normal fs                          | recommended
access file after untar to overlayfs, nfs, docker mount fs    | strongly recommended
file daily usages                                             | no-needed, the sync managed by os
access file in pheriphrals: u-disk, embedded eqpt, mount card | recommended (avoid power down dataloss)
```

about smb (server message block):
smb is a network file sharing protocol developed by ibm and later enhanced by microsoft,
allowing apps to read and write to file and request services from server programs within a local area network (lan).

<hr>

### # issue in production code
copy onu executable binary to different places and start each of them with different cmdline
setups right after the cp operation, an error occur as follows:

```text
Time End: Sun Mar 30 11:35:10 UTC 2025
ls: cannot access '/var/1/OnuHost': No such file or directory
```

related snippet code where the error occurred (hostfw/dockerfile/onu_standard/generic_example.sh):

```text
...
# create a directory for each ONU and copy all files to it
onu_pos=1
for onu in $onus; do
    mkdir -p /var/onu_${onu_pos}
    cp -rf ./* /var/onu_${onu_pos}/      # use the same binary file for each onu by file copy
    onu_pos=$((onu_pos + 1))
done

# loop through each ONU entry in the 'onus' array
onu_pos=1
for onu in $onus; do
    ...
    # onu information display
    echo "--- ONU $onu_pos ---"
    echo "  Time Start: $(date)"
    ...
    echo "  File permissions: $(ls -l /var/onu_${onu_pos}/OnuHost)"      # reporting error
    ...
done
...
```

<hr>

### # solution to the problem
add sync right after the onu binary cp operation, to ensure the file in i/o buffer
down to disk, rather than in page cached state, so that the following file exec & access see no problem.

```text
...
# create a directory for each ONU and copy all files to it
onu_pos=1
for onu in $onus; do
    mkdir -p /var/onu_${onu_pos}
    cp -rf ./* /var/onu_${onu_pos}/      # use the same binary file for each onu by file copy
    onu_pos=$((onu_pos + 1))
done

sync

# loop through each ONU entry in the 'onus' array
onu_pos=1
for onu in $onus; do
    ...
    # onu information display
    echo "--- ONU $onu_pos ---"
    echo "  Time Start: $(date)"
    ...
    echo "  File permissions: $(ls -l /var/onu_${onu_pos}/OnuHost)"      # reporting error
    ...
done
...
```
