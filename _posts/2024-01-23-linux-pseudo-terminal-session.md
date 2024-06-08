---
layout: post
title: "pseudo terminal session: /dev/pts/${session_num} (linux, terminal)"
author: "melon"
date: 2024-01-23 20:50
categories: "2024"
tags:
  - linux
  - terminal
---

nothing is actually stored in /dev/pts. this filesystem lives purely in memory.

<hr>

### # testcases
1 display users login info with related pseudo terminal info:
```text
$ who
metung   pts/1        2024-05-10 20:29 (n-20w1pf3ynjn6.int.nokia-sbell.com)
guolinp  pts/10       2023-10-18 17:01 (:1)
guolinp  pts/11       2023-10-18 17:03 (:1)
guolinp  pts/12       2023-10-18 17:04 (:1)
chaowang pts/13       2024-05-11 20:09 (n-21hepf4t2z1g.int.nokia-sbell.com)
metung   pts/15       2024-05-09 15:49 (n-20w1pf3ynjn6.int.nokia-sbell.com)
metung   pts/16       2024-05-09 15:49 (n-20w1pf3ynjn6.int.nokia-sbell.com)
chaowang pts/18       2024-05-11 20:39 (n-21hepf4t2z1g.int.nokia-sbell.com)
metung   pts/20       2024-05-09 15:50 (n-20w1pf3ynjn6.int.nokia-sbell.com)
jiaguoz  pts/8        2023-11-09 22:50 (:16.0)
chaowang pts/21       2024-05-11 20:40 (n-21hepf4t2z1g.int.nokia-sbell.com)
metung   pts/25       2024-05-09 16:42 (n-20w1pf3ynjn6.int.nokia-sbell.com)
metung   pts/29       2024-05-09 19:53 (n-20w1pf3ynjn6.int.nokia-sbell.com)
```

<p style="margin-bottom: 20px;"></p>

2 send hello to current pseudo terminal:
```text
$ echo 'hello' > /dev/pts/1   # send hello to current terminal
hello
```

<p style="margin-bottom: 20px;"></p>

3 send hello to a certain pseudo terminal:
```text
$ echo 'hello' > /dev/pts/16  # send hello to another terminal
```
inside the terminal linked with /dev/pts/16, the hello is printed on screen:
```text
$(/dev/pts/16) hello
```

<p style="margin-bottom: 20px;"></p>

4 listen to the keystroke of certain terminal:
```text
$(origin) cat /dev/pts/16
```

inside the terminal linked with /dev/pts/16, echo something once a second:
```text
$(/dev/pts/16) bash -c 'while true; do echo hello && sleep 1; done'
```

however, in the terminal with cat, nothing printed.  
typically the /dev/pts/16 will be a bit stuck to use, try type characters with each
following a 'enter':
```text
$(/dev/pts/16) a<enter>
$(/dev/pts/16) b<enter>
```

the characters is echoed from the original terminal:
```text
$(origin) ab
```

try type characters 'cdefg' continuously:
```text
$(/dev/pts/16) c
```
not all the characters got pass to the target terminal, with some left randomly.
```text
$(origin) defg
```

ref: https://unix.stackexchange.com/a/93535
