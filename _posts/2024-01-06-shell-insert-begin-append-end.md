---
layout: post
title: "insert at beginning & append at end (shell)"
author: "melon"
date: 2024-01-06 22:45
categories: "2024"
tags:
  - shell
---


### # insert str at the beginning of file
given a alpine linux apk mirror config file content as follows:
```text
$ cat /etc/apk/repositories
https://dl-cdn.alpinelinux.org/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/community
```

we want to insert some str at the beginning of this file,
which means the newly added mirror has higher priority for apk intake than the default one.

first we check whether the following output meet the need (dry-run):
```text
$ echo "https://artifactory-xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main" \
| cat - /etc/apk/repositories
https://artifactory-blr1.int.net.nokia.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/community
```

then actual modify the file by:
```text
$ echo "https://artifactory-xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main" \
| cat - /etc/apk/repositories 2>&1 | tee /etc/apk/repositories
https://artifactory-xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/community

$ cat /etc/apk/repositories
https://artifactory-xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/community
```

we can also insert a str to a file without adding newline:
```text
$ echo -n "# " | cat - /etc/apk/repositories | tee /etc/apk/repositories
# https://artifactory-xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/community

$ cat /etc/apk/repositories
# https://artifactory-xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/community
```
which will skip the self-maintained mirror at ease.

<hr>

### # insert str at the end of file
append somthing at the end of file:
```text
$ echo "https://a_new_mirror/url" >> /etc/apk/repositories

$ cat /etc/apk/repositories
# https://artifactory-xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/main
https://dl-cdn.alpinelinux.org/alpine/v3.18/community
https://a_new_mirror/url
```
