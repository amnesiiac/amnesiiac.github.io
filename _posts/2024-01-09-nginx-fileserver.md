---
layout: post
title: "containerized fileserver setup by nginx (nginx, fileserver)"
author: "melon"
date: 2024-01-09 21:01
categories: "2024"
tags:
  - nginx
  - container
---


### # buildup containerized nginx fileserver
1 project tree
```text
.
├── compose.yml
├── dockerfile.nginx
└── nginx.conf
```

2 compose.yaml
```text
version: '3.3'

services: 
    server:                                          # service named server
        build:
            context: .                               # docker image context
            dockerfile: dockerfile.nginx             # dockerfile name
        image: alpine/fileserver                     # image name
        container_name: alpine_fileserver
        restart: always
        command: [nginx-debug, '-g', 'daemon off;']
        volumes:                                     # mount
            - "./cache:/mirror/downloads:ro"
        ports:                                       # outside 8888 is nat to 80 inside
            - "8888:80"
        extra_hosts:
            - "host.docker.internal:host-gateway"    # access/serve localhost (host)
```

3 dockerfile.nginx
```text
FROM nginx:1.17-alpine

RUN rm /etc/nginx/conf.d/default.conf
COPY ./nginx.conf /etc/nginx/conf.d/mirror.conf
```

4 nginx.conf
```text
server {
    listen *:80;
    server_name _;

    root /mirror/downloads;
    autoindex on;
}
```

<hr>

### # nginx service project buildup steps
1 build up service docker image configured in compose.yaml:
```text
$ docker-compose build
Building mirror
Sending build context to Docker daemon  14.85kB

Step 1/3 : FROM alpine:3.18
 ---> 8ca4688f4f35
Step 2/3 : RUN apk update &&     apk add gcc make perl wget rsync --no-cache
 ---> Running in 45ea0a0dac46
...

Step 1/3 : FROM nginx:1.17-alpine
 ---> 89ec9da68213
Step 2/3 : RUN rm /etc/nginx/conf.d/default.conf
 ---> Using cache
 ---> de4206e1ef40
Step 3/3 : COPY ./nginx.conf /etc/nginx/conf.d/mirror.conf
 ---> 1dd8118a3509
Successfully built 1dd8118a3509
Successfully tagged mirror_server:latest
```

2 run the fileserver by nginx in background:
```text
$ docker-compose up -d server
Creating mirror_server_1 ... done
```

3 the server will be service on 0.0.0.0:8888, accept all request from any ip addr.
```text
$ docker ps -a | grep server
CONTAINER ID  IMAGE              COMMAND     CREATED  STATUS  PORTS                                  NAMES
ae4fd...      alpine/fileserver  "nginx..."  11s ago  Up 9s   0.0.0.0:8888->80/tcp, :::8888->80/tcp  alpine_fileserver
```

<hr>

### # enable containers on host to access service exposed on localhost (3 methods)
1 add option --add-host to docker run:
```text
docker run -d --add-host host.docker.internal:host-gateway ${image}
```

2 add config in docker-compose.yaml:
```text
extra_hosts:
    - "host.docker.internal:host-gateway"
```

3: add option --network when container startup (make the container share netns with host):
```text
docker run -d --network=host ${image}
```
ref: https://stackoverflow.com/a/24326540

<hr>

### # nginx fileserver as a local mirror for alpine linux apk mirror
ref: https://wiki.alpinelinux.org/wiki/How_to_setup_a_Alpine_Linux_mirror
