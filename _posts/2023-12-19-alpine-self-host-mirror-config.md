---
layout: post
title: "error when init self-hosted apk mirror (alpine)"
author: "melon"
date: 2023-12-19 21:43
categories: "2023"
tags:
  - alpine
  - linux
---

### # reproduce problem
using rebornlinux/devkit/base docker image as env for bootstrap.sh to buildup the base infra,
during manual trigger the bootstrap stage job init-ppc64, error occurs:
```text
$ git config --global url."https://gitlab-ci-token:${CI_JOB_TOKEN}@xxxx.com".insteadOf "https://xxxxx.com"

$ rm -f ~/logs/$CI_JOB_NAME.log

$ bootstrap.sh ppc64 gccgo,norust,nokernel | tee -a ~/logs/$CI_JOB_NAME.log
+ sudo apk update
fetch https://xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
WARNING: updating and opening https://xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main:
         No such file or directory
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/community/x86_64/APKINDEX.tar.gz
v3.18.5-56-g3f4dddfa57b [https://dl-cdn.alpinelinux.org/alpine/v3.18/main]
v3.18.5-55-ge1f11b947c0 [https://dl-cdn.alpinelinux.org/alpine/v3.18/community]
2 unavailable, 0 stale; 20069 distinct packages available
ERROR: Job failed: exit code 1
```

check it locally by starting up a base container, result in same issue:
```text
$(host) docker run -it --rm rebornlinux/base:latest bash

$(alpine) sudo apk update
fetch https://xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
WARNING: updating and opening https://xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main:
         No such file or directory
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/community/x86_64/APKINDEX.tar.gz
v3.18.5-56-g3f4dddfa57b [https://dl-cdn.alpinelinux.org/alpine/v3.18/main]
v3.18.5-55-ge1f11b947c0 [https://dl-cdn.alpinelinux.org/alpine/v3.18/community]
2 unavailable, 0 stale; 20069 distinct packages available

$(alpine) echo $?
2
```

<hr>

### # attempts and comparison analysis
a warning shows: no such file/directory for self-hosted url,
modify /etc/apk/repositories for a comparison analysis:
```text
$ cat /etc/apk/repositories
https://xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/community
```

temporarily delete the self-host url:
```text
$ sudo vi /etc/apk/repositories
https://dl-cdn.alpinelinux.org/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/community
<ctrl+z>
[1]+  Stopped                 sudo vi /etc/apk/repositories
```

try update again (need export http_proxy=http://10.144.1.10:8080) -> succeed
```text
$ sudo apk update
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/community/x86_64/APKINDEX.tar.gz
v3.18.5-56-g3f4dddfa57b [https://dl-cdn.alpinelinux.org/alpine/v3.18/main]
v3.18.5-55-ge1f11b947c0 [https://dl-cdn.alpinelinux.org/alpine/v3.18/community]
OK: 20069 distinct packages available
```

get vi editor back & edit it again (to check if place the url to last works)
```text
$ fg %1
https://dl-cdn.alpinelinux.org/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/community
https://xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main
```

the command failed again
```text
$ sudo apk update
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/community/x86_64/APKINDEX.tar.gz
fetch https://xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
WARNING: updating and opening https://xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main:
         No such file or directory
v3.18.5-56-g3f4dddfa57b [https://dl-cdn.alpinelinux.org/alpine/v3.18/main]
v3.18.5-55-ge1f11b947c0 [https://dl-cdn.alpinelinux.org/alpine/v3.18/community]
2 unavailable, 0 stale; 20069 distinct packages available
```

<hr>

### # root cause
open the self-hosted url: https://xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main, which reported:
```text
{
    "errors": [
        {
            "status": 404,
            "message": "File not found."
        }
    ]
}
```
then update from alpine 3.17 to alpine 3.18, result in a 404 url,
and for apk repo, none 404 url is allowed (in any place).

try using curl + RESTful api to setup a corresponding url (empty), try the steps above,
still the same failure.

as a conclusion, when init an self-hosted mirror for customized apks, we need not null
self-hosted mirror url as a startup.

for more details why this fail, or hack it, see official repo: apk-tools apk update command.
