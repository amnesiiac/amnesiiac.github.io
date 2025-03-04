---
layout: post
title: "fuse: filesystem in userspace (linux, filesys, bypass)"
author: "melon"
date: 2025-03-04 21:51
categories: "2025"
tags:
  - linux
  - todo
---

### # introduction for fuse
todo

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

### # toy code for basic custom fuse setup

```text
#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>

static const char* hello_str = "hello world!\n";
static const char* hello_path = "/hello";

static int hello_getattr(const char* path, struct stat* stbuf){
    int res = 0;
    memset(stbuf, 0, sizeof(struct stat));
    if(strcmp(path, "/") == 0){
        stbuf->st_mode = S_IFDIR | 0755;
        stbuf->st_nlink = 2;
    }
    else if(strcmp(path, hello_path) == 0){
        stbuf->st_mode = S_IFREG | 0444;
        stbuf->st_nlink = 1;
        stbuf->st_size = strlen(hello_str);
    }
    else{
        res = -ENOENT;
    }
    return res;
}

static int hello_readdir(const char* path, void* buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info* fi){
    (void) offset;
    (void) fi;
    if(strcmp(path, "/") != 0){
        return -ENOENT;
    }
    filler(buf, ".", NULL, 0);
    filler(buf, "..", NULL, 0);
    filler(buf, hello_path + 1, NULL, 0);
    return 0;
}

static int hello_open(const char* path, struct fuse_file_info* fi){
    if(strcmp(path, hello_path) != 0){
        return -ENOENT;
    }
    if((fi->flags & 3) != O_RDONLY){
        return -EACCES;
    }
    return 0;
}

static int hello_read(const char* path, char* buf, size_t size, off_t offset, struct fuse_file_info* fi){
    size_t len;
    (void) fi;
    if(strcmp(path, hello_path) != 0){
        return -ENOENT;
    }
    len = strlen(hello_str);
    if(offset < len){
        if(offset + size > len){
            size = len - offset;
        }
        memcpy(buf, hello_str + offset, size);
    }
    else{
        size = 0;
    }
    return size;
}

static struct fuse_operations hello_oper = {
    .getattr  = hello_getattr,
    .readdir  = hello_readdir,
    .open     = hello_open,
    .read     = hello_read,
};

int main(int argc, char* argv[]){
    return fuse_main(argc, argv, &hello_oper);
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

### # fuse implementation for board eeprom
todo

<hr>

### # reference
1 fuse golang implementation: https://github.com/bazil/fuse  
2 fuse c implementation: libfuse (userspace clib): https://github.com/libfuse/libfuse/tree/master  
3 fuse kernel module: fuse.ko
