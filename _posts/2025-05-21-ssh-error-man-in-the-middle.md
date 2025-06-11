---
layout: post
title: "solution to ssh warning: man in the middle (ssh)"
author: "melon"
date: 2025-05-21 18:21
categories: "2025"
tags:
  - ssh
---

this article explains "man in the middle" warning of a known ssh connection on your host machine,
and provide basic steps to get rid of it.

<hr>

### # man in the middle warning poblem
setup ssh connection from host to hostfw container (host machine port 16022 -> iptables nat rules -> container#1 port 822),
which works promising.

```txt
        ┌────────────────────────────────────┐                ┌────────────────────────────────────┐
        │  wrapper container                 │                │  wrapper container                 │
        │                                    │                │                                    │
        │  ┌─────────────┐  ┌─────────────┐  │                │  ┌─────────────┐  ┌─────────────┐  │
        │  │ container#1 │  │ container#2 │  │ update iptable │  │ container#1 │  │ container#2 │  │
        │  │    (822)    │  │    (822)    │  │ -------------> │  │    (822)    │  │    (822)    │  │
        │  └──────│──────┘  └──────|──────┘  │   nat rules    │  └──────|──────┘  └──────│──────┘  │
        │         │                |         │                │         |                │         │
        │         iptables-nat-rules         │                │         iptables-nat-rules         │
        │                 │                  │                │                 │                  │
        └─────────────────│──────────────────┘                └─────────────────│──────────────────┘
                       (16022)    host machine                               (16022)    host machine
```

by del conntrack caches (sudo conntrack -D) & replace nat table rules (replace container#1-ip:822 with container#2-ip:822),
we try setup new connection towards the redun service (ssh root@0 -p 16022), we get warnings from ssh as following text:

```text
$ ssh admin@0 -p 50004
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the RSA key sent by the remote host is
SHA256:shzatimAdJDl8qeLHV2dK4UfA1EhZbRrg7W1QHRdgZs.
Please contact your system administrator.
Add correct host key in /home/metung/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /home/metung/.ssh/known_hosts:79
Password authentication is disabled to avoid man-in-the-middle attacks.
Keyboard-interactive authentication is disabled to avoid man-in-the-middle attacks.

Permission denied (publickey,password).
```

<hr>

### # root cause analysis
before and after we modify the container nat rules, the ssh connection path is changed,
thus the ssh-key for newly-setup connection is recomputed and conflict with stored key for
first setup connection.

<hr>

### # solution
just delete all history key caches related to this ssh connection:

```text
$ ssh-keygen -R [0]:50004      # rm all stored entries for first ssh connection (ip:port & key pair)
$ ssh-keygen -R [0]:49977      # rm all stored entries for second ssh connection (ip:port & key pair)
```

retry the ssh connection via 50004 or 49977, which are both fine:

```text
$ ssh admin@0 -p 50004
admin@0's password:

B:admin@FAD-Chassis#   <--- login succeed
```

```text
$ ssh admin@0 -p 49977
admin@0's password:

B:admin@FAD-Chassis#   <--- login succeed
```
