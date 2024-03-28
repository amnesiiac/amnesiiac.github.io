---
layout: post
title: "search in vim editor (vim)"
author: "melon"
date: 2024-01-26 20:48
categories: "2024"
tags:
  - vim
---

### # search in vim with regex pattern used
```text
$ vim file_to_search 
abuild-3.11.1.tar.gz              curl-8.4.0.tar.xz           libcap-2.69.tar.xz
acl-2.3.1.tar.gz                  CVE-2021-43618.patch        libcap-ng-0.8.3.tar.gz
alpine-conf-3.16.2.tar.gz         fakeroot_1.31.orig.tar.gz   libedit-20221030-3.1.tar.gz
apk-tools-v2.14.0.tar.gz          file-5.45.tar.gz            libev-4.33.tar.gz
argon2-20190702.tar.gz            fortify-headers-1.1.tar.gz  libffi-3.4.4-2.tar.gz
attr-2.5.1.tar.gz                 gcc-12-20220924.tar.xz      libidn2-2.3.4.tar.gz
binutils-2.40.tar.xz              gmp-6.2.1.tar.xz            libmd-1.0.4.tar.xz
brotli-1.0.9.tar.gz               isl-0.26.tar.bz2            gc
busybox-1.36.1.tar.bz2            json-c-0.16.tar.gz          gc
ca-certificates-20230506.tar.bz2  kmod-30.tar.xz              libucontext-1.2.tar.xz
c-ares-1.19.1.tar.gz              lddtree-1.27.tar.gz         libxml2-2.11.6.tar.xz
cryptsetup-2.6.1.tar.gz           gc                          linux-6.3.tar.xz
CUnit-2.1-3.tar.bz2               libbsd-0.11.7.tar.xz        LVM2.2.03.21.tgz
```

inside vim editor, type the following:
```text
:g/^lib.*tar.gz/gc
```
which search for all instance start with lib, with uncertain num of char in the middle,
end with tar.gz. `.` matches any character, `*` matches any times of previous character.

<hr>

### # search pattern within a range of lines
```text
/\%>9l\%<33llib.*tar.gz
```
which search for all instance in lines range (12,24) that matches lib.*tar.gz pattern.  
the following help page illustrates the usages.
```text
:help search-range
:help /\%>l
```

another method to search a pattern within a range of lines is as:
```text
:10,32s/lib.*tar.gz//gc
```
simply press no to just navigate each occurence of each match within range, the range is only
valid for the frist loop.

<hr>

### # search for all lines without matching certain pattern
match all the lines that does not contain character:
```text
/^\(\(.*foo.*\)\@!.\)*$
```

match all the lines that does not contain certain pattern:
```text
/^\(\(.*lib.*tar.gz.*\)\@!.\)*$
```

<hr>

### # log filter
show only lines that does not mattch the specified pattern on screen:
```text
:v/lib.*tar.gz/p
```

in contrast, show only lines that match the specified pattern:
```text
:g/lib.*tar.gz/p
```

<hr>

### # filter huge logs with regex & redirect to new log file
inside a vim editor, enter the following to create & edit a new copy of current file: junk.log
```text
:sav junk.log
```

delete all lines that match the given pattern:
```text
:g/lib.*tar.gz/d
```
then use :wq to exit, finally the output junk.log file will contains only 'interested' lines.

<hr>

### # add suitable command couple to nvim macro (todo)

<hr>

### # reference
https://vim.fandom.com/wiki/Search_for_lines_not_containing_pattern_and_other_helpful_searches
