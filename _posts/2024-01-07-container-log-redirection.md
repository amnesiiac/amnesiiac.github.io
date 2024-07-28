---
layout: post
title: "container log listening & redirection (container)"
author: "melon"
date: 2024-01-07 21:38
categories: "2024"
tags:
  - container
---

### # issus using shell for log listening & redirection
given a nginx container started by nerdctl cli:

```text
$(nerdctl alpine container) nerdctl ps -a
CONTAINER ID  IMAGE         COMMAND                  CREATED     STATUS  PORTS                  NAMES
82d31060b102  nginx:latest  "/docker-entrypoint..."  6 days ago  Up      0.0.0.0:49155->80/tcp  nginx-82d31
```

in shell env, the following log redirection operation failed:

```text
$(nerdctl alpine container) nerdctl logs -ft 82d31060b102 &>> ./log
sh: syntax error: unexpected redirection
```

<hr>

### # solution: use bash for log listening & redirection
the above issue can be solved by using bash instead:

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

the nerdctl alpine container setup can refer to containerized-nerdctl (github) for guidance.

<hr>

### # log listener & redirector in python
the following provides an example illustrating how to listen & redirect container logs:

```text
def logs_to_file(self, file):
    self.log_file = file
    sh.run("{} logs -ft {} &>> {}".format(self.container_cli, self.id, self.log_file))
    return 0
```

after device reaches ready state, the platform hostfw will continue listening the docker log output,
and redirect the log to workdir.
