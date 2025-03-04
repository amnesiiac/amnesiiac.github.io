---
layout: post
title: "rm -rf failed even with root privilege (linux, container, lsof)"
author: "melon"
date: 2023-12-26 20:21
categories: "2023"
tags:
  - container
---

this article describe an situation when try rm -rf to cleanup a mounted workdir of a running container instance,
the rm operation failed even with root privilege.

a) ssh to the remote server.  
b) startup the hostfw simulation instance.  
c) no operation on the server for minutes, keep the tty idle until the ssh connection session is down.  
d) ssh to the server again.  
e) try rm the files under the workdir assigned:

```text
$ rm -rf /repo/metung/workdir
rm: cannot remove '/repo/metung/workdir/setup_ihub_traffic_xxx/.nfs00000000631aad3c0000147f': Device or resource busy
```

f) check if hostfw simulation instance still existing by state of wrapper container:

```text
$ docker ps -a | grep setup_ihub_traffic_xxx
c0b98ea292ac  hostfw-local.artifactory-espoo1.int.net.xxxxx.com/release/clicktest_dind:v6.1metung
"/sbin/tini -- docke…"  47 minutes ago  Up 47 minutes  2375-2376/tcp ... :::33351->63003/tcp  setup_ihub_traffic_xxx
```

g) use lsof to check if any process is working on the nfs resouce:

```text
$ lsof /repo/metung/workdir/setup_ihub_traffic_xxx/.nfs00000000631aad3c0000147f
lsof: WARNING: can't stat() nsfs file system /run/docker/netns/default
    Output information may be incomplete.
lsof: WARNING: can't stat() fuse.gvfsd-fuse file system /run/user/211158/gvfs
    Output information may be incomplete.
lsof: WARNING: can't stat() overlay file system /repo/docker/overlay2/5abe44cdec.../merged
    Output information may be incomplete.
lsof: WARNING: can't stat() nsfs file system /run/docker/netns/6d970ebd697a
    Output information may be incomplete.
lsof: WARNING: can't stat() overlay file system /repo/docker/overlay2/dc375af4b8.../merged
    Output information may be incomplete.
lsof: WARNING: can't stat() nsfs file system /run/docker/netns/35bcae4689a7
    Output information may be incomplete.
lsof: WARNING: can't stat() overlay file system /repo/docker/overlay2/970c7ddc3b.../merged
    Output information may be incomplete.
lsof: WARNING: can't stat() nsfs file system /run/docker/netns/de032af7a164
    Output information may be incomplete.
lsof: WARNING: can't stat() fuse.gvfsd-fuse file system /run/user/39510/gvfs
    Output information may be incomplete.
lsof: WARNING: can't stat() overlay file system /repo/docker/overlay2/cea2bf41f9.../merged
    Output information may be incomplete.
lsof: WARNING: can't stat() nsfs file system /run/docker/netns/310bc8b73ad7
    Output information may be incomplete.
lsof: WARNING: can't stat() overlay file system /repo/docker/overlay2/3b29ed042a.../merged
    Output information may be incomplete.
lsof: WARNING: can't stat() nsfs file system /run/docker/netns/75b243909c8e
    Output information may be incomplete.
lsof: WARNING: can't stat() fuse.gvfsd-fuse file system /run/user/3002146/gvfs
    Output information may be incomplete.

COMMAND  PID      USER    FD  TYPE  DEVICE  SIZE/OFF        NODE NAME
vim      2954463  metung  4u  REG   0,52    4096 1662692668 /repo/metung/workdir/setup_ihub_traffic_xxx/.nfs0000...0147f
```

h) kill the process & try remove the stuff again, the operation executed sucessfully.

```text
$ kill -sigkill 2954463
$ rm -rf /repo/metung/workdir
$ echo $?
0
```

<hr>

### # extra desc about lsof
lsof can be used to display info about files that are open by processes.
it can be particularly useful for debugging, monitoring system activity, and troubleshooting file-related issues.

```text
$ lsof                         # list all open files
$ lsof -u ${username}          # list open files of a specified user
$ lsof -p ${pid}               # list open files of a specified process
$ lsof -c ${command}           # list open files of a specified command
$ lsof +D ${path_to_dir}       # list open files under certain directory
$ lsof ${path_to_file}         # list all processes attached to certain file
$ lsof -i                      # list all network connections
$ lsof -t ...                  # list all open files of certain type
```
