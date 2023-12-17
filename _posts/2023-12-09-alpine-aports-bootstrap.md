---
layout: post
title: "alpine/aports/scripts/bootstrap.sh (alpine)"
author: "melon"
date: 2023-12-09 12:19
categories: "2023"
tags:
  - alpine
---

### # bootstrap.sh
This script creates a local cross-compiler, and uses it to cross-compile an Alpine Linux base system for new architecture.  
ref: https://gitlab.alpinelinux.org/alpine/aports/-/blob/master/scripts/bootstrap.sh?ref_type=heads 

<hr>

### # problems when using bootstrap.sh to cross compile base alpine linux system
config proxy before using apk (enable access to external network to reach all apk source), 
see setup proxy inside gitlab-runner for details:
```text
# upper / lower case for compatiable for docker & wget & curl...
$ export https_proxy=http://10.144.1.10:8080
$ export http_proxy=http://10.144.1.10:8080
$ export HTTPS_PROXY=http://10.144.1.10:8080
$ export HTTP_PROXY=http://10.144.1.10:8080
```

remove pre-installed apks & built-up dirs in rebornlinux/devkit/Dockerfile.toolchain container:
```text
# keep the env clean
$ sudo rm -rf sysroot-*
$ sudo apk del gcc-ppc gcc-go-ppc gcc-ppc64 gcc-go-ppc64 gcc-mips64 gcc-aarch64
```
  
invoke the bootstrap script inside toolchain image container:
```text
$ aports/scripts/bootstrap.sh ppc64 gccgo,norust,nokernel <--- take this as a case
...
(30/30) Installing .hostdepends-fakeroot (20231206.135836)
OK: 270 MiB in 30 packages
>>> fakeroot: Cleaning up srcdir
>>> fakeroot: Cleaning up pkgdir
>>> fakeroot: Fetching https://deb.debian.org/debian/pool/main/f/fakeroot/fakeroot_1.29.orig.tar.gz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0   260    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (22) The requested URL returned error: 404
>>> ERROR: fakeroot: fetch failed
>>> fakeroot: Uninstalling dependencies...
(1/22) Purging .makedepends-fakeroot (20231206.135834)
(2/22) Purging build-base-ppc64 (0.5-r3)
...
```

<hr>

### # solution
by errno 404 (file not found), we could found available one on pkg host, update local info as:
```text
$ cd ~/aports/main/fakeroot
$ vim ./APKBUILD             # modify pkgver to existed one on https://deb.debian.org/debian/pool/main/f/fakeroot/
$ abuild checksum            # automatically modify APKBUILD hash validation value
>>> fakeroot: Fetching https://deb.debian.org/debian/pool/main/f/fakeroot/fakeroot_1.32.2.orig.tar.gz
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                               Dload  Upload   Total   Spent    Left  Speed
100  557k  100  557k    0     0   223k      0  0:00:02  0:00:02 --:--:--  223k
>>> fakeroot: Updating the sha512sums in APKBUILD...
```

finally, the 404 not found err can be temporarily solved.  
for a specific version of aports, all pkg is only available for a duration, but after that, the reliance packages used for building-up the system base is no longer hosted on upstream's ftp site.

thus, we should build the base of certain release of our crafted system within the 'valid duration', then make them persistent on artifactory.
