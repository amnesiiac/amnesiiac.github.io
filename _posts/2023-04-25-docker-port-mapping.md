---
layout: post
title: "port mapping && port publishing (docker)"
author: "twistfatezz"
date: 2023-04-25 22:01
categories: "2023" 
tags:
  - container
  - ongoing
---
{% raw %}
### # check port mapping of containers
```text
$ docker ps --format="table {{.ID}}\t{{.Image}}\t{{.Ports}}"
CONTAINER ID   IMAGE       PORTS
fff4712570ae   transhell   
f8f45cf255d2   blog:v1     0.0.0.0:4000->4000/tcp
4802b8612083   nvim:v1 
```
the network interface 4000 of host is mapping to the container's 4000/tcp inner port.

<hr>

### # check port mapping of images
```text
$ docker inspect --format="{{json .Config.ExposedPorts}}" blog:v1
null
```
the null output due to no "EXPOSE ${port}" in blog:v1 Dockerfile, even the running container is configured with port mapping dynamically.

<hr>

### # exposing ports
ports can be exposed in Dockerfile:
```text
FROM httpd:latest
EXPOSE 80
```
this will let the applications inside container to publish/listen on port 80.  

<hr>

### # publishing ports
setup a jekyll blog service through container using port mapping:
```text
$ docker run -it -d --rm -p 4000:4000 -v ${v1}:${v2} blog:v1 bundle exec jekyll serve --host 0.0.0.0
```
the "-p" flag can be used without pointing to a host port:
```text
$ docker run -it -d --rm -p 80 httpd:latest
```
then the container inner port 80 will bind to a random port on host, and its docker daemon's duty to avoid duplications cross instances. <br>
the "-p" flag support bind mount to a network interface:
```text
$ docker run -it -d --rm -p 127.0.0.1:80:80 httpd:latest
```
then the httpd will only listen to the loopback port 80.

<hr>

### # publishing all ports
to publish all the port exposed in dockerfile, use:
```text
$ docker run -it -d --rm --publish-all httpd:latest
```
this will align random port on host to map with. <br>
if combine the publish all with "-p", the latter will overwrite the random port allocated:
```text
$ docker run -it -d --rm --publish-all -p 127.0.0.1:80:80 httpd:latest
```

<hr>

### # inter-container communication (ongoing: test it)
containers that share a network can always communicate with each other, even if their ports have not been explicitly published:
```text
$ docker network create demo-network
$ docker run -d --network demo-network --name web web:latest
$ docker run -d --network demo-network --name database database:latest
```
the web container could connect to a mysql server runnning at some available ports.

<hr>

### # publishing range of ports
sample-1: using --publish-all will assign 100 random port mapping inside to 8000-8100.
```text
# Dockerfile
FROM httpd:latest
EXPOSE 8000-8100
```
```text
$ docker run --publish-all
```
sample-2: explicitly bing a range of ports to a container range as normal.
```text
$ docker run -p 6000-6100:8000-8100
```
bind a large range of ports will impact on performances due to creating iptable rules for each.


{% endraw %}
