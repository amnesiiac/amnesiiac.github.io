---
layout: post
title: "subshell usages (shell)"
author: "melon"
date: 2023-12-26 21:49
categories: "2023"
tags:
  - shell
---

### # use subshell for group command execution logic
false/true clause represent some cmd that might raise failure or success.
we want to echo msg according to cmd running status, and the whole clause return 1 if the cmd fails.
below are some attempts:

attempt 1: unexpected result
```text
$ false || echo 'err msg'; false      # pre cmd fail, print err msg
err msg
$ echo $?
1

$ true || echo 'err msg'; false       # pre cmd succeed, dont print err msg, whole cmd should return 0
$ echo $?
1                                     # unexpected
```

attempt 2: unexpected result
```text
$ false || echo 'err msg' && false    # pre cmd fail, print err msg
err msg
$ echo $?
1

$ true || echo 'err msg' && false     # pre cmd succeed, dont print err msg, whole cmd should return 0
$ echo $?
1                                     # unexpected
```

attempt 3: expected result
```text
$ false || (echo 'err msg' && false)  # pre cmd fail, print err msg
err msg
$ echo $?
1

$ true || (echo 'err msg' && false)   # expected
$ echo $?
0
```

<hr>

### # code in action
rebornlinux/aports/.gitlab-ci.yml
