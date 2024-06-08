---
layout: post
title: "docker registry mirror for acceleration (docker)"
author: "melon"
date: 2024-01-08 20:38
categories: "2024"
tags:
  - container
---

list of docker image mirror sites:
```text
docker official mirror:    https://registry.docker-cn.com
daocloud mirror:           http://f1361db2.m.daocloud.io
azure china mirror:        https://dockerhub.azk8s.cn
tstc mirror:               https://docker.mirrors.ustc.edu.cn        (*)
aliyum mirror:             https://${docker_id}.mirror.aliyuncs.com
qiniu cloud:               https://reg-mirror.qiniu.com
wangyi 163:                https://hub-mirror.c.163.com
tencent:                   https://mirror.ccs.tencentyun.com
```

config the mirror for docker daemon:
```text
$ cat /etc/docker/daemon.json
{
    ...
    "registry-mirrors": [
        "https://dockerhub.azk8s.cn",
        "https://mirror.ccs.tencentyun.com"
    ],
    ...
}
```

restart the docker daemon to make it work:
```text
$ mkdir -p /etc/systemd/system/docker.service.d
$ systemctl daemon-reload
$ systemctl enable docker
$ systemctl restart docker
```

confirm whether the configuration take effect:
```text
$ docker info | grep -C 2 Mirrors
Insecure Registries:
  hubproxy.docker.internal:5555
  127.0.0.0/8
Registry Mirrors
  https://dockerhub.azk8s.cn
  https://mirror.ccs.tencentyun.com
```
