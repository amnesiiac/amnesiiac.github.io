---
layout: post
title: "linux distribution: alpine (linux)"
author: "melon"
date: 2023-08-18 22:07
categories: "2023"
tags:
  - alpine
  - linux
  - ongoing
---

### # creating an alpine linux pkg repo (create a new repo for custom pkg)
(1) setup the repo  
\> decide where you want the repository to live, typically at /repo/  
\> place the package files in the repository in a directory named for the appropriate architecture, e.g. /repo/x86_64/  
(a repository can be a simple directory on the server itself, or be hosted at an http, https, or ftp location)

(2) placing the apk under /repo/x86_64/

(3) indexing the packages under dir
```text
$ apk index -o /repo/x86_64/APKINDEX.unsigned.tar.gz /repo/x86_64/*.apk  # generated unsigned index using custom apks
```

(4) copy the unsigned index to index
```text
$ cp /repo/x86_64/APKINDEX.unsigned.tar.gz /repo/x86_64/APKINDEX.tar.gz
```

(5) regist custom repo to your public alpine apk repo:
```text
$ vim /etc/apk/repositories                               # add "/repo/" inside
```
```text
$ cat /etc/apk/repositories
https://dl-cdn.alpinelinux.org/alpine/v3.18/main          # priority-1
https://dl-cdn.alpinelinux.org/alpine/v3.18/community     # priority-2
/repo/                                                    # priority-3 to 1
```

(6) testing the unsigned repo 
```text
comment out any other repos in /etc/apk/repositories
$ apk update
WARNING: Ignoring /repo/x86_64/APKINDEX.tar.gz: UNTRUSTED signature
OK: 27 distinct packages available              # 27 is not the num of custom repo pkgs, but pre-installed system pkgs)
```

```text
$ apk update --allow-untrusted                  # re-run the update cmd && allow it to use un-trusted repo
OK: 38 distinct packages available              # extra 11 pkg in custom repo is added
$ apk search ${package_name} --allow-untrusted  # search for pkg name to confirm the pkgs are available
```

now we constructed an un-trustd & valid alpine linux repo

(7) signing the un-trusted one
```text
generating key pairs
$ apk add abuild
$ abuild-keygen -a -i                           # interactive set the key path
```

sign the custom repo apkindex with the full path of the private key created above
```text
$ abuild-sign -k ~/alpine-devel@example.com-5629d7e6.rsa /repo/x86_64/APKINDEX.tar.gz
```

(8) testing the signed repo
```text
$ apk update                                    # no longer need the "un-trusted" flag
OK: 38 distinct packages available
```
uncommented the above /etc/apk/repositories and enjoy.

<hr>

### # apk indexing
```text
apk-tools 2.14.0, compiled for x86_64.

usage: apk index [<OPTIONS>...] PACKAGES...

Description:
    apk index creates a repository index from a list of package files. See
    repositories for more information on repository indicies.

Global options:
    -f, --force           Enable selected --force-* options (deprecated)
    -i, --interactive     Ask confirmation before performing certain
                          operations
    -p, --root ROOT       Manage file system at ROOT
    -q, --quiet           Print less information
    -U, --update-cache    Alias for '--cache-max-age 1'
    -v, --verbose         Print more information (can be specified twice)
    -V, --version         Print program version and exit
    -X, --repository REPO
                          Specify additional package repository
    --allow-untrusted     Install packages with untrusted signature or no
                          signature
    --arch ARCH           Temporarily override architecture, to be combined
                          with --root
    --cache-dir CACHEDIR  Temporarily override the cache directory
    --cache-max-age AGE   Maximum AGE (in minutes) for index in cache before
                          it's refreshed
    --force-binary-stdout
                          Continue even if binary data will be printed to the
                          terminal
    --force-broken-world  Continue even if WORLD cannot be satisfied
    --force-missing-repositories
                          Continue even if some of the repository indexes are
                          not available
    --force-non-repository
                          Continue even if packages may be lost on reboot
    --force-old-apk       Continue even if packages use unsupported features
    --force-overwrite     Overwrite files in other packages
    --force-refresh       Do not use cached files (local or from proxy)
    --keys-dir KEYSDIR    Override directory of trusted keys
    --no-cache            Do not use any local cache path
    --no-check-certificate
                          Do not validate the HTTPS server certificates
    --no-interactive      Disable interactive mode
    --no-network          Do not use the network
    --no-progress         Disable progress bar even for TTYs
    --print-arch          Print default arch and exit
    --progress            Show progress
    --progress-fd FD      Write progress to the specified file descriptor
    --purge               Purge modified configuration and cached packages
    --repositories-file REPOFILE
                          Override system repositories, see repositories
    --timeout TIME        Timeout network connections if no progress is made
                          in TIME seconds
    --wait TIME           Wait for TIME seconds to get an exclusive repository
                          lock before failing

Index options:
    -d, --description TEXT
                          Add a description to the index
    --merge               Merge PACKAGES into the existing INDEX                 (*) merging
    -o, --output FILE     Output generated index to FILE
    --prune-origin        Prune packages from the existing INDEX with same
                          origin as any of the new PACKAGES during merge
    -x, --index INDEX     Read an existing index from INDEX to speed up the
                          creation of the new index by reusing data when
                          possible
    --no-warnings         Disable the warning about missing dependencies
    --rewrite-arch ARCH   Set all package's architecture to ARCH
  
For more information: man 8 apk-index
```

<hr>

### # code in action (reborn linux)
todo
