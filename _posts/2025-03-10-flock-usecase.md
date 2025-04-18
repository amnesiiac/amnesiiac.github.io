---
layout: post
title: "flock usecases (flock, shell, sync)"
author: "melon"
date: 2025-03-10 22:24
categories: "2025"
tags:
  - shell
  - sync
---

```text
#!/bin/bash

lock_and_unlock() {
    exec 200>./tmp/flock.lock                                           # take fd 200 for writing to lock file
    flock -x 200 && echo "[$BASHPID]: exclusive lock acquired"          # exclusive flock on writing to 200
    echo "[$BASHPID]: doing some critical section stuff..." && sleep 5  # critical section
    ls -l /proc/"$BASHPID"/fd                                           # check fd info for $BASHPID (pid of current jobs)
    flock -u 200 && echo "[$BASHPID]: released exclusive lock"          # unlock
}

lock_and_unlock &
lock_and_unlock &
wait
```

execute above script:

```text
$ bash t.sh
[28453]: exclusive lock acquired
[28453]: doing some critical section stuff...
total 0
lr-x------ 1 metung metung 64 Apr 16 14:54 0 -> /dev/null
lrwx------ 1 metung metung 64 Apr 16 14:54 1 -> /dev/pts/11
lrwx------ 1 metung metung 64 Apr 16 14:54 2 -> /dev/pts/11
l-wx------ 1 metung metung 64 Apr 16 14:54 200 -> /repo1/metung/txt/alcoholism/sync/flock/tmp/flock.lock
[28453]: released exclusive lock
[28454]: exclusive lock acquired
[28454]: doing some critical section stuff...
total 0
lr-x------ 1 metung metung 64 Apr 16 14:54 0 -> /dev/null
lrwx------ 1 metung metung 64 Apr 16 14:54 1 -> /dev/pts/11
lrwx------ 1 metung metung 64 Apr 16 14:54 2 -> /dev/pts/11
l-wx------ 1 metung metung 64 Apr 16 14:54 200 -> /repo1/metung/txt/alcoholism/sync/flock/tmp/flock.lock
[28454]: released exclusive lock
```

<hr>

### # explanations for `exec 200>./tmp/flock.lock`
exec basic usages:
if command is specified, it replaces the shell, no new process is created.
if command is not specified, any redirections take effect in the current shell

exec `200>${custom_file}` inside t.sh will set `${custom_file}` as the target when writing on fd (200)
in the current process of t.sh.
