---
layout: post
title: "using self-hosted artifactory for alpine apks (alpine)"
author: "melon"
date: 2023-12-06 22:31
categories: "2023"
tags:
  - alpine
  - linux
  - ongoing
---

### # problem when setup self-maintained alpine mirror
the problem is that: some apk (for cross compiling & part of buildroot) cannot be added using 'apk add xxx', relying on self-maintained repo.

<hr>

### # steps to check & fix
inside docker container of toolchain, check the groups:
```text
$(centos7) docker run -it --rm rebornlinux/toolchain bash
$(reborn)  groups
reborn wheel abuild
```

the apk repo source is configured as:
```text
$ cat /etc/apk/repositories
https://artifactory.xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.17/main # <- self-hosted
https://dl-cdn.alpinelinux.org/alpine/v3.17/main
https://dl-cdn.alpinelinux.org/alpine/v3.17/community
```

on self-hosted remote ftp service from Jfrog, the dir is as:  
```text
Index of rebornlinux-generic-local/alpine/v3.17/main
Name     Last modified      Size
../
aarch64/  24-Apr-2023 14:53    -
mips64/   24-Apr-2023 14:56    -
ppc/      24-Apr-2023 14:57    -
ppc64/    24-Apr-2023 15:01    -
x86_64/   24-Apr-2023 15:05    -
```

try refresh according to repo source configured:  
```text
$ sudo apk update
fetch https://artifactory.xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.17/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.17/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.17/community/x86_64/APKINDEX.tar.gz
main  [https://artifactory.xxxxx.com/artifactory/rebornlinux-generic-local/alpine/v3.17/main]
v3.17.5-190-g6351cf9e666 [https://dl-cdn.alpinelinux.org/alpine/v3.17/main]
v3.17.5-189-g8f10631eb0e [https://dl-cdn.alpinelinux.org/alpine/v3.17/community]
OK: 17849 distinct packages available
```

check global apk list of all configured source to findout all build-base available:
```text
$ apk list | grep build-base
build-base-ppc64-0.5-r3 noarch {build-base-ppc64} (MIT)
build-base-ppc-0.5-r3 noarch {build-base-ppc} (MIT)
build-base-mips64-0.5-r3 noarch {build-base-mips64} (MIT)
build-base-0.5-r3 x86_64 {build-base} (MIT) [installed]     <--- installed by default
build-base-aarch64-0.5-r3 noarch {build-base-aarch64} (MIT)
```
the installation of x86_64 arch build-base can be confirmed by:
```text
$ apk info | grep build-base
build-base
```

try to install build-base-aarch64-0.5-r3 (in self-maintained mirror), raise err:
```text
$ sudo apk add build-base-aarch64-0.5-r3
ERROR: unable to select packages:
build-base-aarch64-0.5-r3 (no such package):
  required by: world[build-base-aarch64-0.5-r3]
```

try to install with shortened name, found it in index but package not found:
```text
$ sudo apk add build-base-aarch64
(1/3) Installing libstdc++-dev-aarch64 (12.2.1_git20220924-r4)
(2/3) Installing g++-aarch64 (12.2.1_git20220924-r4)
(3/3) Installing build-base-aarch64 (0.5-r3)
ERROR: build-base-aarch64-0.5-r3: package mentioned in index not found (try 'apk update')
Executing busybox-1.35.0-r29.trigger
1 error; 990 MiB in 90 packages
```

check & modify APKBUILD file for it (/aport/main/$arch/buildbase/APKBUILD):
```text
# Maintainer: Natanael Copa <ncopa@alpinelinux.org>
pkgname=build-base
pkgver=0.5
pkgrel=3
url=http://dev.alpinelinux.org/cgit
pkgdesc="Meta package for build base"
depends="binutils file gcc g++ make libc-dev fortify-headers patch"
if [ "$CHOST" != "$CTARGET" ]; then
      pkgname="$pkgname-$CTARGET_ARCH"
      pkgdesc="$pkgdesc ($CTARGET_ARCH)"
      depends="binutils-$CTARGET_ARCH gcc-$CTARGET_ARCH g++-$CTARGET_ARCH $depends"
fi
arch="noarch"   <--- change this to arch="all"
license="MIT"
options="!check"

build() {
      :
}

package() {
      mkdir -p "$pkgdir"
}
```

finally, compile it using abuild and upload the built APK to self-maintained repo:
the upload script could be found at rebornlinux/devkit/scripts/deploy.sh,
then try sudo apk add, will succeed on adding a build-base from different arch.

<hr>

### # code in action
todo: rebornlinux/aport

<hr>

### # reference
ref: https://jfrog.atlassian.net//si/jira.issueviews:issue-html/RTFACT-26820/RTFACT-26820.html
