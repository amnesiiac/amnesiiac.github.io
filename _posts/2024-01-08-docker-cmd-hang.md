---
layout: post
title: "docker cli command hang problem (docker)"
author: "melon"
date: 2024-01-08 19:38
categories: "2024"
tags:
  - container
---

### # issue
when container framework startup, the progress hangs after the following log printed out:
```text
[17:28:59     INFO]: device[setup_lt_1] copy user sshkey finished
```
the program failed to continue, with no log output anymore.

<hr>

### # analysis & debug
enable strace for detection of hostfw framework hang problem, the logs is as:
```text
...
[17:28:56     INFO]: dev[setup_lt_1] : init
[17:28:56     INFO]: dev(setup_lt_1),set status : init
[17:28:56     INFO]: dev(setup_lt_1), workdir = /repo/xxx/workdir/setup_lt_1
[17:28:56     INFO]: builder create start
[17:28:56     INFO]: create dev : setup_lt_1
[17:28:56     INFO]: device[setup_lt_1] creating now ....
[17:28:56     INFO]: Check docker images exist: clicktest_dind:v5.1
) = 0 (Timeout)
wait4(1701822, 0x7ffc4ef01cf4, WNOHANG, NULL) = 0
select(0, NULL, NULL, NULL, {tv_sec=2, tv_usec=0}
[17:28:57     INFO]: Found docker images and pull: hostfw-remote.xxx.com/release/clicktest_dind:v5.1
[17:28:57     INFO]: self.image.persistent_src: [None] and self.persistent_dir:[/repo/xxx/workdir/setup_lt_1/persistent]
[17:28:57     INFO]: device[setup_lt_1] add mounts ....
[17:28:57     INFO]: Exporting SFPs mapping files...
) = 0 (Timeout)
wait4(1701822, 0x7ffc4ef01cf4, WNOHANG, NULL) = 0
select(0, NULL, NULL, NULL, {tv_sec=2, tv_usec=0}
[17:28:59     INFO]: device[setup_lt_1] create interface ....
[17:28:59     INFO]: create_net_intf name: eth0, type: docker0_port
[17:28:59     INFO]: new netcard : eth0, type : docker0_port
[17:28:59     INFO]: device[setup_lt_1] copy user sshkey finished
) = 0 (Timeout)
wait4(1701822, 0x7ffc4ef01cf4, WNOHANG, NULL) = 0
select(0, NULL, NULL, NULL, {tv_sec=2, tv_usec=0}) = 0 (Timeout)
wait4(1701822, 0x7ffc4ef01cf4, WNOHANG, NULL) = 0
select(0, NULL, NULL, NULL, {tv_sec=2, tv_usec=0}) = 0 (Timeout)
wait4(1701822, 0x7ffc4ef01cf4, WNOHANG, NULL) = 0
select(0, NULL, NULL, NULL, {tv_sec=2, tv_usec=0}) = 0 (Timeout)
wait4(1701822, 0x7ffc4ef01cf4, WNOHANG, NULL) = 0
...
```

<hr>

### # root cause
in the vm machine that startup hostfw, in which the following container are dead:
```text
96364729c302   host-image-dfmb-c-4047683  ykang-dfmb-c-1-ho7M
dd0f506cc8f2   host-image-dfmb-c-4032741  ykang-dfmb-c-0-0s7H
```
these containers failed to response for any docker cli cmds,
need restart docker daemon to clean.

inside hostfw, the docker inspect cmd is trying to check mounts from a container,
but the related syscall poll received no response.
{% raw %}
```text
status, text = sh.run("{} inspect -f '{{.Mounts}}' {}".format(container_cli, id))
```

add wait timeout to fix:
```text
status, text = sh.run("{} inspect -f '{{.Mounts}}' {}".format(container_cli, id), wait_timeout=3)
```
{% endraw %}
