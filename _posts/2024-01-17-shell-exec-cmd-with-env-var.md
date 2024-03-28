---
layout: post
title: "execute cmd with env variable set (shell)"
author: "melon"
date: 2024-01-17 21:50
categories: "2024"
tags:
  - shell
---

### # code in action: rebornlinux/devit/buildrepos.sh
this script use buildrepo script of alpinelinux to generate common packages
host on alpine mirror.  
ref: https://mirrors.alpinelinux.org/  
ref: https://wiki.alpinelinux.org/wiki/Abuild_and_Helpers#buildrepo
```text
#!/bin/bash

# if param $1 is set, then do nothing, else print help
[ -z "$1" ] && (echo "Usage: $0 <target_arch>"; exit 1)
target_arch=$1
repos="main xxxxx community"

echo "Build [$repos] for $target_arch begin"
for repo in $repos; do
    echo "Build $repo for $target_arch begin"

    # <<< invoke a cmd with env var set for it >>>
    CHOST=$target_arch /usr/bin/buildrepo -k -a ${CI_PROJECT_DIR} -l \
          ~/logs/$CI_JOB_NAME-$CI_PIPELINE_ID/$(date +"%Y-%m-%d_%H-%M-%S") "$repo"
    echo "Build $repo for $target_arch end"
done
echo "Build [$repos] for $target_arch end"
```

<hr>

### # code in action: rebornlinux/devkit/build.sh
```text
# build apk and dep, save logs, checkapk, classify into aport_ok/aport_ng
build_aport() {
    local repo="$1" aport="$2"
    cd "$APORTSDIR/$repo/$aport"

    # <<< cross compilation using abuild with CHOST env set, and using if to check cmd status >>>
    if CHOST="$ABUILD_ARCH" abuild -r 2>&1 | report "build-$aport"; then
        checkapk | report "checkapk-$aport" || true
        aport_ok="$aport_ok $repo/$aport"
    else
        aport_ng="$aport_ng $repo/$aport"
    fi
}
```

<hr>

### # shell test tricks for sentence with cmd
to judge whether a combination of shell sentence work as assumed to be,
using true & false to mimic the cmd to invoke:
```text
# sentence to test: if msg=hello $(some_cmd_executation); then echo 0; else echo 1; fi

$ if msg=hello true; then echo 0; else echo 1; fi
0
$ if msg=hello false; then echo 0; else echo 1; fi
1
```
