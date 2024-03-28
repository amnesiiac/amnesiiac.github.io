---
layout: post
title: "redirection op: > and >> (shell)"
author: "melon"
date: 2024-01-06 21:45
categories: "2024"
tags:
  - shell
---

### # difference between > and \>>
◆ use operation \>> to redirect logs to non-existed file  
```text
$ for i in {1..10}; do echo "installing packages" >> test 2>&1; sleep 1; done
$ cat test
installing packages
installing packages
installing packages
installing packages
installing packages
installing packages
installing packages
installing packages
installing packages
installing packages
```

◆ use operation > to redirect logs to non-existed file
```text
$ for i in {1..10}; do echo "installing packages" > test 2>&1; sleep 1; done
$ cat test
installing packages
```

◆ use operation \>> to redirect logs to existed file  
using \>> to echo 10 fake log into an existed file:
```text
$ cat test
installing packages

$ for i in {1..10}; do echo "installing packages" >> test 2>&1; sleep 1; done

$ cat test | wc -l
11
```

◆ use operation > to redirect logs to existed file  
using > to echo 10 fake log into an existed file (tail won't start if file non-existed):
```text
$ cat test
installing packages

$ for i in {1..10}; do echo "installing packages" > test 2>&1; sleep 1; done

$ cat test
installing packages
```

retry the above settings with tail listening the file test output:
```text
$ cat test
installing packages

$ tail -f ./test | grep -e '^install' &
[1] 23709
$ installing packages

press <enter>

$ jobs
[1]+  Running                 tail -f ./test | grep --color=auto -e '^install' &

$ for i in {1..10}; do echo "installing packages" > test 2>&1; sleep 1; done
tail: test: file truncated
installing packages
tail: test: file truncated
installing packages
tail: test: file truncated
installing packages
tail: test: file truncated
installing packages
tail: test: file truncated
installing packages
tail: test: file truncated
installing packages
tail: test: file truncated
installing packages
...
```

<hr>

### # implementation analysis
todo: dive into c implementation of > and \>> to get clear about: why syscall truncate
is invoked when using > repetitively with an existed file.

<hr>

### # code in action (rebornlinux/aport/.gitlab-ci.yml)
◆ Test1: examine behavior of > to redirect ci/cd logs:
```text
.init:
  before_script:
    - rm -f ~/logs/$CI_JOB_NAME.log; touch ~/logs/$CI_JOB_NAME.log
    - tail -f ~/logs/$CI_JOB_NAME.log | grep -e ">>>" &             # continuously listening on log file
  when: manual
  stage: bootstrap
  timeout: 6 hours

init-x86_64:
  extends: .init
  script:
    - echo 'running bootstrap.sh'
    - for i in {1..100}; do echo ">>> installing packages available" > ~/logs/$CI_JOB_NAME.log 2>&1; sleep 1; done
  tags:
    - aport-specific
```

gitlab ci/cd job log is as:
```text
$ rm -f ~/logs/$CI_JOB_NAME.log && touch ~/logs/$CI_JOB_NAME.log
$ tail -f ~/logs/$CI_JOB_NAME.log | grep -e "retries left" -e "fetch http" -e "packages available" &
$ echo 'running bootstrap.sh'
running bootstrap.sh

$ for i in {1..100}; do echo "installing packages available" > ~/logs/$CI_JOB_NAME.log 2>&1; sleep 1; done
installing packages available  # grep only 1 line output
Job succeeded
```
which is weird to have only 1 line filtered out.
<br/>

◆ Test2: examine the behavior of \>> to redirection ci/cd logs:
```text
.init:
  before_script:
    - rm -f ~/logs/$CI_JOB_NAME.log; touch ~/logs/$CI_JOB_NAME.log
    - tail -f ~/logs/$CI_JOB_NAME.log | grep -e ">>>" &              # continuously listening on log file
  when: manual
  stage: bootstrap
  timeout: 6 hours

init-x86_64:
  extends: .init
  script:
    - echo 'running bootstrap.sh'
    - for i in {1..100}; do echo ">>> installing packages available" >> ~/logs/$CI_JOB_NAME.log 2>&1; sleep 1; done
  tags:
    - aport-specific
```

gitlab ci/cd job log:
```text
$ rm -f ~/logs/$CI_JOB_NAME.log && touch ~/logs/$CI_JOB_NAME.log
$ tail -f ~/logs/$CI_JOB_NAME.log | grep -e ">>>" &
$ echo 'running bootstrap.sh'
running bootstrap.sh

$ for i in {1..100}; do echo "installing packages available" >> ~/logs/$CI_JOB_NAME.log 2>&1; sleep 1; done
installing packages available
installing packages available
installing packages available
installing packages available
installing packages available
...
Job succeeded
```
which seems satisfying our need.
