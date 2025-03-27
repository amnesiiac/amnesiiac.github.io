---
layout: post
title: "rust env setup utilities (rust intro)"
author: "melon"
date: 2025-01-06 21:25
categories: "2025"
tags:
  - rust
  - docker
---

1 dockerfile for rust env setup:

```text
FROM alpine:3.18

ENV https_proxy=http://10.144.1.10:8080
ENV http_proxy=http://10.144.1.10:8080

# rust-src: rust std src code context for ide/analysis; rust-clippy: linter; rustfmt: formator
RUN apk update && \
    apk add sudo && \
    apk add rust && apk add cargo && \
    apk add rust-src && \
    apk add rust-clippy && \
    apk add rustfmt && \
    rm -rf /var/cache/apk/*

# add user 1013, add user to wheel, enable user sudo without pwd
RUN adduser -u 1013 -D metung&& \
    adduser metung wheel && \
    echo 'metung ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers

# switch to use metung (install go for it)
USER metung

COPY entrypoint.sh /usr/bin/
ENTRYPOINT [ "sh", "entrypoint.sh"]
```

2 entrypoint.sh:

```text
#!/bin/sh

exec "$@"
```

3 info: context for docker image buildup.

```text
http_proxy = http://10.158.100.9:8080
https_proxy = http://10.158.100.9:8080
dockerfile = dockerfile
image_name = rustdev
tag = latest
```

4 makefile: automation script for docker image buildup.

```text
include info

build:
	docker build -f ${dockerfile} -t $(image_name):$(tag) --build-arg HTTP_PROXY=$(http_proxy) \
--build-arg HTTPS_PROXY=$(https_proxy) .

default: build
```

5 r.sh: project env container startup script.

```text
#!/bin/sh

# project name at this folder
[ -z "$1" ] && { echo 'no project provided!'; exit 1; }
proj="$1"

script_dir="$(realpath $(dirname "$proj"))"
image='rustdev:latest'
cmd='sh'

docker run -it --rm \
    -w /"$proj" \
    -v "$script_dir":/"$proj" \
    -v /var/run/docker.sock:/var/run/docker.sock:rw \
    -v /var/run/netns:/var/run/netns:shared \
    --user "$(id -u)":"$(id -g)" \
    --cap-add=CAP_SYS_ADMIN \
    --cap-add=NET_ADMIN \
    --security-opt apparmor=unconfined \
    --pid=host \
    --privileged \
    $image $cmd

# --pid=host: enable /proc system in container the same as host
# --privileged: enable operations on the container proc system: open / unshare ...

# to enable shared docker in container with host
# -v /var/run/docker.sock:/var/run/docker.sock:rw
# -v /var/run/netns:/var/run/netns:shared
# --cap-add=CAP_SYS_ADMIN \
# --cap-add=NET_ADMIN \
# --security-opt apparmor=unconfined \
```
