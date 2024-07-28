---
layout: post
title: "access container netns from host machine (docker, netns)"
author: "melon"
date: 2024-01-08 20:38
categories: "2024"
tags:
  - container
---

{% raw %}

the following steps shows: how to access & operate on certain container's netns in host machine  
1 get the pid of the container:

```text
$ docker ps
$ pid=$(docker inspect -f '{{.State.Pid}}' ${container_id})
```

2 create softlink from procfs to runtime dir on host machine forcibly.

```text
$ mkdir -p /var/run/netns
$ ln -sfT /proc/${pid}/ns/net /var/run/netns/${container_id}
```

3 execute command in the container netns:

```text
$ ip netns exec ${container_id} ip a
```

4 appendix: why softlink to /var/run/netns provide the access to the netns of the container process?  
by convention a named network namespace is an object at /var/run/netns/NAME that can be opened.
the fd resulting from opening /var/run/netns/NAME refers to the specified network namespace.
holding that fd open keeps the network namespace alive.
the fd can be used with the setns(2) system call to change the network namespace associated with a task.

ref: https://man7.org/linux/man-pages/man8/ip-netns.8.html

{% endraw %}
