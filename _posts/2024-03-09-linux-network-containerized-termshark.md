---
layout: post
title: "termshark: a containerized tui tool like wireshark (network, pkt analysis)"
author: "melon"
date: 2024-03-09 20:02
categories: "2024"
tags:
  - network
  - container
---

this article focus on a terminal user-interface for tshark, inspired by wireshark.  

<hr>

### # buildup containerized termshark
1 dockerfile
```text
FROM alpine:3.18

ENV https_proxy=http://10.144.1.10:8080                     # container proxy
ENV http_proxy=http://10.144.1.10:8080

RUN apk update && \
    apk add bash && \
    apk add termshark && \
    apk add net-tools && \
    rm -rf /var/apk/cache/*

COPY termshark.toml /root/.config/termshark/termshark.toml  # override config
```

2 customized termshark.toml:
```text
[main]
color-tsharks = ['/usr/bin/tshark']
dark-mode = false
last-used-tshark = '/usr/bin/tshark'
packet-colors = false
term = 'xterm-256color'
theme-256 = 'solarized'
validated-tsharks = ['/usr/bin/tshark']
```

3 makefile for docker image buildup automation:
```text
img:=$$(basename $$(pwd))

build:
	docker build -t $(img) -f dockerfile .

clean:
	docker rmi $(img):latest

default: build
```
run make to buildup a docker image contains termshark at ease.  
4 start a default bridge mode termshark env, test termshark functionalities:
```text
$(host) docker run -it --rm termshark:latest sh
$(ts env) ping -I lo 127.0.0.1 >/dev/null 2>&1 &   # sliently send icmp packets to lo
$(ts env) termshark -i lo                          # capture & analysis pkts using termshark
```

<hr>

### # termshark container usecases
1 inspect interfaces belong to host netns:
```text
$(host) docker run -it --rm --privileged=true --net=host --name termshark termshark:latest sh
```

2 inspect interfaces belong to host container netns:
```text
$(host) docker run -it --rm --privileged=true --network=container:${host_containername} --name termshark termshark sh
```

3 inspect interfaces of any container in certain netns:  
```text
$(root) nsenter --target ${real_pid_of_container} --net bash
$(root) termshark -i ${interface}
```

containerized termshark still cannot share netns with one created by user: /var/run/netns/${uuid}:
```text
$ docker run -it --rm --net=/var/run/netns/${uuid} termshark sh
```

so we have to maintain the termshark available as executable bin, enter the netns, then test accordingly.

<hr>

### # appearance of termshark on windows12 mobaxterm

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/ts.pdf" width="800"/>

<hr>

### # reference
ref: https://github.com/gcla/termshark
