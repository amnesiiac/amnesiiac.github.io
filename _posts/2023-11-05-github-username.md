---
layout: post
title: "github account related (github)"
author: "melon"
date: 2023-11-05 20:03
categories: "2023"
tags:
  - github
---

### # issue when git clone after rename github account username
after changing github username, try add ssh pub key to github account:
```text
$ cat ~/.ssh/id_rsa.pub | pbcopy
```
add the pub key to github account, then try clone a repo already existed on remote:
```text
$ git clone git@github.com:melonterminator/amnesia.git
```
```text
cloning into 'amnesia'...
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the RSA key sent by the remote host is
SHA256:uNiVztksCsDhcc0u9e8BujQXVUpKZIDTMczCvj3tD2s.
Please contact your system administrator.
Add correct host key in /Users/mac/.ssh/known_hosts to get rid of this message.
Offending RSA key in /Users/mac/.ssh/known_hosts:4
RSA host key for github.com has changed and you have requested strict checking.
Host key verification failed.
fatal: unable to read the remote repository.

please confirm that you have the correct access right and the repository exists.
```
the username is changed, so need to remove local obsolete one:
```text
$ ssh-keygen -R github.com
```
```text
# Host github.com found: line 4
/Users/mac/.ssh/known_hosts updated.
Original contents retained as /Users/mac/.ssh/known_hosts.old
```
try git clone again to automactically add ECDSA of github.com to known hosts:
```text
$ git clone git@github.com:melonterminator/amnesia.git
```
```text
cloning into 'amnesia'...
The authenticity of host 'github.com (20.205.243.166)' can't be established.
ECDSA key fingerprint is SHA256:p2QAMXNIC1TJYWeIOttrVc98/R1BUFWu3/LiyKgUfQM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'github.com' (ECDSA) to the list of known hosts.
Warning: the ECDSA host key for 'github.com' differs from the key for the IP address '20.205.243.166'
Offending key for IP in /Users/mac/.ssh/known_hosts:6
Are you sure you want to continue connecting (yes/no)? yes
remote: Enumerating objects: 61, done.
remote: Counting objects: 100% (61/61), done.
remote: Compressing objects: 100% (51/51), done.
remote: Total 61 (delta 23), reused 48 (delta 10), pack-reused 0
accepting objects: 100% (61/61), 46.22 KiB | 178.00 KiB/s, completed.
dealing delta: 100% (23/23), completed.
```
