---
layout: post
title: "container log listening & redirection (container)"
author: "melon"
date: 2024-01-07 21:38
categories: "2024"
tags:
  - container
---

### # issue: shell log listener
given a nginx container started by nerdctl cli:
```text
$(nerdctl alpine container) nerdctl ps -a
CONTAINER ID  IMAGE         COMMAND                  CREATED     STATUS  PORTS                  NAMES
82d31060b102  nginx:latest  "/docker-entrypoint..."  6 days ago  Up      0.0.0.0:49155->80/tcp  nginx-82d31
```

in shell env, the following log redirection operation will result in err:
```text
$(nerdctl alpine container) nerdctl logs -ft 82d31060b102 &>> ./log
sh: syntax error: unexpected redirection
```

<hr>

### # solution: bash log listener
to fix this, simply switch to bash:
```text
$(nerdctl alpine container) apk add bash
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/community/x86_64/APKINDEX.tar.gz
(1/1) Installing bash (5.2.15-r5)
Executing bash-5.2.15-r5.post-install
Executing busybox-1.36.1-r2.trigger
OK: 570 MiB in 88 packages

$(nerdctl alpine container) bash                                    # switch to bash
$(nerdctl alpine container) nerdctl logs -ft 82d31060b102 &>> .log  # try listening container log & redirection
^C
```
the nerdctl alpine container can be setup using github repo: containerized_nerdctl.
<hr>

### # code in action (an example illustrate how to listen & redirect container logs)
after device reaches ready state, the hostfw will hang on listening the docker log output & redirect to workdir
```text
def logs_to_file(self, file):
    self.log_file = file
    sh.run("{} logs -ft {} &>> {}".format(self.container_cli, self.id, self.log_file))
    return 0
```
ref: hostfw/pkg_host/sandbox/sb_docker.py
