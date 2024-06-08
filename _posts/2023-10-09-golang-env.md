---
layout: post
title: "containerized golang compilation env (golang)"
author: "melon"
date: 2023-10-09 21:57
categories: "2023"
tags:
  - golang
  - container
---


### # buildup containerized golang compilation env
1 dockerfile
```text
FROM golang:1.22.1-alpine3.18

# proxy
ENV https_proxy=http://10.144.1.10:8080
ENV http_proxy=http://10.144.1.10:8080

# musl-dev: https://stackoverflow.com/a/42366740/10515951
RUN apk update && \
    apk add bash && \
    apk add make && \
    apk add git && \
    apk add musl-dev && \
    apk add build-base && \
    apk add docker && \
    apk add vim && \
    rm -rf /var/apk/cache/*

# enable cgo for cross compilation (need build-base)
ENV CGO_ENABLED="1"

# golang debugger (need vim rather than vi)
ENV DELVE_EDITOR=vim
RUN git clone https://github.com/go-delve/delve && \
    cd delve && \
    go install github.com/go-delve/delve/cmd/dlv && \
    cd -

COPY entrypoint.sh /usr/bin/
ENTRYPOINT [ "entrypoint.sh" ]

# apk add tini
# ENTRYPOINT [ "/sbin/tini", "--", "entrypoint.sh" ]
# [WARN  tini (939)] Tini is not running as PID 1 and isn't registered as a child subreaper.
# Zombie processes will not be re-parented to Tini, so zombie reaping won't work.
# To fix the problem, use the -s option or set the environment variable TINI_SUBREAPER to
# register Tini as a child subreaper, or run Tini as PID 1.
```

2 entrypoint.sh
```text
#!/bin/sh

exec "$@"
```

3 makefile variable config file (info):
```text
http_proxy = http://10.158.100.9:8080
https_proxy = http://10.158.100.9:8080
dockerfile = dockerfile
image_name = golang
tag = alpine3.18
```

4 makefile for automation of docker image buildup
```text
include info

build:
	docker build -f ${dockerfile} -t $(image_name):$(tag) \
        --build-arg HTTP_PROXY=$(http_proxy) --build-arg HTTPS_PROXY=$(https_proxy) .

default: build
```

5 an example startup script of golang env container for nicon project:
```text
#!/bin/sh

# the project name
proj='nicon'
script_dir="$(realpath $(dirname "$proj"))"
image='golang:alpine3.18'
cmd='bash'

# startup golang compilation container with docker tool enabled and 
docker run -it --rm \
    -w /"$proj" \
    -v "$script_dir":/"$proj" \
    -v /var/run/docker.sock:/var/run/docker.sock:rw \
    -v /var/run/netns:/var/run/netns:shared \
    --cap-add=CAP_SYS_ADMIN \
    --cap-add=NET_ADMIN \
    --pid=host \
    --privileged \
    $image $cmd
```

enable /proc system in container the same as host:
```text
--pid=host
```

enable operations on the container proc system: open/unshare \...
```text
--privileged  # include the apparmor=unconfined
```

enable shared dockerd between the docker in container and on host:
```text
-v /var/run/docker.sock:/var/run/docker.sock:rw
-v /var/run/netns:/var/run/netns:shared
--cap-add=CAP_SYS_ADMIN \
--cap-add=NET_ADMIN \
--security-opt apparmor=unconfined \
```
