---
layout: post
title: "opengrok resources (site/local)"
author: "twistfatezz"
date: 2023-05-13 09:18
categories: "2023"
tags:
  - opengrok
---

### # linux source code
The Dockerfile:
```text
FROM opengrok/docker:1.7

ARG V=6.0  # kernel version

RUN mkdir -p /opengrok/src
RUN cd /opengrok/src \
&& curl https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/snapshot/linux-${V}.tar.gz \
   --output linux.tar.gz \
&& tar -xvf linux.tar.gz && rm linux.tar.gz

# Trigger opengrok indexer manually
# RUN opengrok-indexer \
#       -J=-Djava.util.logging.config.file=/opengrok/etc/logging.properties \
#       -a /opengrok/lib/opengrok.jar -- \
#       -c /usr/local/bin/ctags \
#       -s /opengrok/src -d /opengrok/data -H -P -S -G \
#       -W /opengrok/etc/configuration.xml -U http://localhost:8080/

WORKDIR /usr/local/tomcat
EXPOSE 8080
CMD ["/scripts/start.py"]
```
build up the docker images:
```text
docker build -t linux:v6.0 .
```
start opengrok indexing inside container, and publish/listen service on localhost:
```text
docker run -d -p 127.0.0.1:8080:8080 kernel6.0
```
check the indexing process by:
```text
docker logs ${container_id}
```

<hr>

### # libhv source code
The Dockerfile:
```text
FROM opengrok/docker:1.7

ARG V=1.3.1  # libhv version

RUN apt update && \
    apt install -y git
RUN mkdir -p /opengrok/src && \
    cd /opengrok/src && \
    git clone https://github.com/ithewei/libhv.git

# Trigger opengrok indexer manually
# RUN opengrok-indexer \
#       -J=-Djava.util.logging.config.file=/opengrok/etc/logging.properties \
#       -a /opengrok/lib/opengrok.jar -- \
#       -c /usr/local/bin/ctags \
#       -s /opengrok/src -d /opengrok/data -H -P -S -G \
#       -W /opengrok/etc/configuration.xml -U http://localhost:8080/

WORKDIR /usr/local/tomcat
EXPOSE 8080
CMD ["/scripts/start.py"]
```
build up docker images:
```text
docker build -t libhv .
```
start opengrok indexing inside container, and publish/listen service on localhost:
```text
docker run -d -p 127.0.0.1:8080:8080 libhv
```
check the indexing process by:
```text
docker logs ${container_id}
```
However, if we start the container using other ports, the service will not start correctly:
```shell
docker run -d -p 127.0.0.1:6666:8080 ${images}
```

<hr>

### # usefully opengrok online sites for source code
• [tensorflow source code](https://opengrok.szdev.com/tensorflow/)  
• [pytorch source code](https://opengrok.szdev.com/pytorch/)  
• [tvm source code](https://opengrok.szdev.com/tvm/)  
• [llvm source code](https://opengrok.szdev.com/llvm/)  
• [CUDA samples](https://opengrok.szdev.com/cuda/)  
• [android source code](https://opengrok.szdev.com/android/)  

