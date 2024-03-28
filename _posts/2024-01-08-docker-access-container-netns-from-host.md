---
layout: post
title: "access container netns from host (docker)"
author: "melon"
date: 2024-01-08 20:38
categories: "2024"
tags:
  - container
---

### # how to access & operate on container netns in host
get the pid of certain container:
{% raw %}
```text
$ docker ps
$ pid=$(docker inspect -f '{{.State.Pid}}' ${container_id})
```
{% endraw %}

create softlink from procfs netns to runtime dir on host forcibly,
always treat the generated link as a file.
```text
$ mkdir -p /var/run/netns
$ ln -sfT /proc/${pid}/ns/net /var/run/netns/${container_id}
```

execute command in the container netns:
```text
$ ip netns exec ${container_id} ip a
```
