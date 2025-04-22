---
layout: post
title: "fuse: filesystem in userspace (linux, filesys, bypass)"
author: "melon"
date: 2025-03-04 21:51
categories: "2025"
tags:
  - linux
---

todo: basic intro to fuse filesystem

<hr>

### # dockerfile for env of fuse toy code execution

```text
FROM alpine:3.18

# proxy
ENV https_proxy=http://10.144.1.10:8080
ENV http_proxy=http://10.144.1.10:8080

# musl-dev: https://stackoverflow.com/a/42366740/10515951
RUN apk update && \
    apk add gcc && \
    apk add make && \
    apk add vim && \
    apk add sudo && \
    apk add musl-dev && \
    apk add fuse && \
    apk add fuse-dev && \
    rm -rf /var/apk/cache/*

# add user 1013, add user to wheel, enable user sudo without pwd
RUN adduser -u 1013 -D metung&& \
    adduser metung wheel && \
    echo 'metung ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers

# switch to use metung (install go for it)
USER metung

# annotations
# fuse: for -lfuse
# fuse-dev: for enable /usr/include/fuse.h
# musl-dev: for enable #include <sys/types.h> in /usr/include/fuse/fuse_common.h
```

<hr>

### # toy code for custom fuse fs setup

```text
#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>

static const char* hello_str = "hello world!\n";                 // content of hello file
static const char* hello_path = "/hello";                        // path to hello file

static int hello_getattr(const char* path, struct stat* stbuf){  // handle getattr
    int res = 0;
    memset(stbuf, 0, sizeof(struct stat));                       // clear mem for stat buf
    if(strcmp(path, "/") == 0){                                  // if path is root dir /
        stbuf->st_mode = S_IFDIR | 0755;                         // set st_mod 0755 of a dir
        stbuf->st_nlink = 2;                                     // set hard link to it as 2 (from its parent dir & itself .)
    }
    else if(strcmp(path, hello_path) == 0){                      // if path is hello file path /hello
        stbuf->st_mode = S_IFREG | 0444;                         // set st_mod 0444 of a regular file
        stbuf->st_nlink = 1;                                     // set hard link of regular file as 1 (from its parent dir)
        stbuf->st_size = strlen(hello_str);                      // set st_size as the size of hello str
    }
    else{
        res = -ENOENT;                                           // for other path, return no file or dir error
    }
    return res;
}

static int hello_readdir(const char* path, void* buf,
                         fuse_fill_dir_t filler, off_t offset,
                         struct fuse_file_info* fi){             // handle readdir (e.g. ls /hello)
    (void) offset;                                               // suppress compiler warnings for unused
    (void) fi;
    if(strcmp(path, "/") != 0){                                  // check if path is /, or ret no such file or dir
        return -ENOENT;
    }
    filler(buf, ".", NULL, 0);                                   // fill dir buf with cur dir .
    filler(buf, "..", NULL, 0);                                  // fill dir buf with parent dir ..
    filler(buf, hello_path + 1, NULL, 0);                        // fill dir buf with file path (exclude leading /)
    return 0;
}

static int hello_open(const char* path, struct fuse_file_info* fi){
    if(strcmp(path, hello_path) != 0){                           // check if path is hello_path, or ret no entry err
        return -ENOENT;
    }
    if((fi->flags & 3) != O_RDONLY){                             // check if read-only mode, or ret perm denied
        return -EACCES;
    }
    return 0;
}

static int hello_read(const char* path, char* buf, size_t size,
                      off_t offset, struct fuse_file_info* fi){  // handle read
    size_t len;
    (void) fi;
    if(strcmp(path, hello_path) != 0){
        return -ENOENT;
    }
    len = strlen(hello_str);                                     // set max available len of bytes for read
    if(offset < len){
        if(offset + size > len){
            size = len - offset;
        }
        memcpy(buf, hello_str + offset, size);                   // set buf by read result
    }
    else{
        size = 0;                                                // size read is 0
    }
    return size;                                                 // return size of byte read
}

static struct fuse_operations hello_oper = {    // define supported op, so the kernel req can invoke them accordingly
    .getattr  = hello_getattr,                  // called when file attr are requsted
    .readdir  = hello_readdir,                  // called when dir content is requested
    .open     = hello_open,                     // called when file opened
    .read     = hello_read,                     // called when file content is requested
};

int main(int argc, char* argv[]){
    return fuse_main(argc, argv, &hello_oper);  // fuse_main: entry for fusefs, init fs & start eventloop
}
```

<hr>

### # compilation utilities
1 info: makefile env variable file.

```text
http_proxy = http://10.158.100.9:8080
https_proxy = http://10.158.100.9:8080
dockerfile = dockerfile
image = alpinefuse
tag = latest
```

2 makefile: the core script for fuse toy code env image buildup & compilation & test & cleanup.

```text
include info

all: fuse

fuse:
	gcc -D_FILE_OFFSET_BITS=64 -DFUSE_USE_VERSION=22 t.c -o mkfuse -lfuse

.PHONY: clean test env

env:
	docker build -f ${dockerfile} -t $(image):$(tag) --build-arg HTTP_PROXY=$(http_proxy) \
		--build-arg HTTPS_PROXY=$(https_proxy) .

test: fuse 
	mkdir -p /tmp/fuse
	./mkfuse /tmp/fuse
	ls -l /tmp/fuse
	cat /tmp/fuse/hello

clean:
	-rm mkfuse
	fusermount -u /tmp/fuse
	-rm -r /tmp/fuse
```

3 r.sh: the start script to launch a env with all toy code present.

```text
#!/bin/sh

# the project name
proj='fuse'
image='alpinefuse'
cmd='sh'

script_dir="$(realpath $(dirname "$proj"))"

docker run -it \
           --rm \
           -w /"$proj" \
           -v "$script_dir":/"$proj" \
           -v /var/run/docker.sock:/var/run/docker.sock:rw \
           --user "$(id -u)":"$(id -g)" \
           --cap-add=SYS_ADMIN \
           --device /dev/fuse \
           --hostname melon \
           "$image" \
           "$cmd"

# annotations:
# --cap-add=SYS_ADMIN: for avoid fusermount permission denied error in container
# --device /dev/fuse: 
# avoid error when trying to: fuse: device not found, try 'modprobe fuse' first
```

<hr>

### # action

```text
$ make fuse
gcc -D_FILE_OFFSET_BITS=64 -DFUSE_USE_VERSION=22 t.c -o mkfuse -lfuse
```

```text
$ make test
gcc -D_FILE_OFFSET_BITS=64 -DFUSE_USE_VERSION=22 t.c -o mkfuse -lfuse
mkdir -p /tmp/fuse
./mkfuse /tmp/fuse
ls -l /tmp/fuse
total 0
-r--r--r--    1 root     root            13 Jan  1  1970 hello
cat /tmp/fuse/hello
hello world!
```

```text
$ make clean
rm mkfuse
fusermount -u /tmp/fuse
rm -r /tmp/fuse
```

<hr>

### # fuse toy code based on libfuse
libfuse api comes in two flavors:  
1 libfuse high-level sync api: incoming req from kernel are passed to fuse mainloop by callback registered.
the high-level callback work with filename or path instead of inode, and the process of request is done when
callback func return.
ref: http://libfuse.github.io/doxygen/hello_8c.html  
2 libfuse low-level async api: incoming req from kernel are passed to fuse mainloop by callback registered.
the low-level callback work with inode and response must be processed with a separate set of apis.
ref: http://libfuse.github.io/doxygen/fuse__lowlevel_8h.html

<hr>

### # reference
1 fuse golang implementation: https://github.com/bazil/fuse  
2 fuse c implementation: libfuse (userspace clib): https://github.com/libfuse/libfuse/tree/master  
3 libfuse doc: http://libfuse.github.io/doxygen/index.html  
4 fuse kernel module: fuse.ko
