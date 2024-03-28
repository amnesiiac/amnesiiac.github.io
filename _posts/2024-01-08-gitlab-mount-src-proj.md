---
layout: post
title: "issues when mount src project into gitlab-runner (gitlab-runner)"
author: "melon"
date: 2024-01-08 22:00
categories: "2024"
tags:
  - gitlab
---

### # background
rebornlinux/aports is cloned from alpine/aports, which act as like buildroot for alpine distributions.

the task is to invoke aports/scripts/bootstrap.sh to generated rootfs & basic apk using cross compilation.

however, an issue happened when try modify aports/packages/main/libffi/APKBUILD and commit to rebornlinux/aports repo,
keep APKBUILD for other apk the same as before.

the desired bootstrap job init-ppc64 output should be like following:
```text
$ ${CI_PROJECT_DIR}/scripts/bootstrap.sh ppc64 gccgo,norust,nokernel > ~/logs/$CI_JOB_NAME.log 2>&1 \
    || (tail -n 100 ~/logs/$CI_JOB_NAME.log; false)
>>> bootstrap-ppc64: Creating sysroot in /home/reborn/sysroot-ppc64/
>>> bootstrap-ppc64: Building cross-compiler
>>> binutils-ppc64: Package is up to date
>>> gcc-ppc64: Package is up to date
>>> build-base-ppc64: Package is up to date
>>> bootstrap-ppc64: Cross building base system
>>> fortify-headers: Package is up to date
>>> linux-headers: Package is up to date
>>> musl: Package is up to date
>>> libc-dev: Package is up to date
>>> pkgconf: Package is up to date
>>> zlib: Package is up to date
>>> openssl: Package is up to date
>>> ca-certificates: Package is up to date
...
>>> libffi: Building main/libffi 3.4.4-r2 (using abuild 3.11.1-r0) started Wed, 27 Dec 2023 06:53:57 +0000
>>> libffi: Checking sanity of /builds/rebornlinux/aports/main/libffi/APKBUILD...
>>> libffi: Analyzing dependencies...
>>> libffi: Installing for build: build-base-ppc64 texinfo
>>> libffi: Installing for host: busybox build-base linux-headers
>>> libffi: Cleaning up srcdir
>>> libffi: Cleaning up pkgdir
...
>>> libffi-doc*: Package size: 80.0 KB
>>> libffi-doc*: Compressing data...
>>> libffi-doc*: Create checksum...
>>> libffi-doc*: Create libffi-doc-3.4.4-r2.apk
>>> libffi*: Tracing dependencies...
>>> libffi*: Package size: 80.0 KB
>>> libffi*: Compressing data...
Job succeeded
```
all unchanged apk are skipped from rebuilding, the newly changed apk libffi should be rebuilt.

however, after we resubmit another commit to aport repo, and test the skip behavior of bootstrap stage again,
the bootstrap cross compilation (from x86_64 to ppc64) rebuild every apk with aport, distfiles, packages, apkcache.
```text
$ ${CI_PROJECT_DIR}/scripts/bootstrap.sh ppc64 gccgo,norust,nokernel > ~/logs/$CI_JOB_NAME.log 2>&1 \
    || (tail -n 100 ~/logs/$CI_JOB_NAME.log; false)
>>> bootstrap-ppc64: Building cross-compiler
>>> binutils-ppc64: Building main/binutils-ppc64 2.40-r7 (using abuild 3.11.1-r0)
>>> binutils-ppc64: Checking sanity of /builds/rebornlinux/aports/main/binutils/APKBUILD...
>>> binutils-ppc64: Analyzing dependencies...
...
>>> binutils-ppc64: Updating the main/x86_64 repository index...
>>> binutils-ppc64: Signing the index...
>>> musl-dev: Building main/musl-dev 1.2.4-r2 (using abuild 3.11.1-r0)
>>> musl-dev: Checking sanity of /builds/rebornlinux/aports/main/musl/APKBUILD...
>>> musl-dev: Analyzing dependencies...
...
>>> musl-dev: Updating the main/ppc64 repository index...
>>> musl-dev: Signing the index...
>>> gcc-pass2-ppc64: Building main/gcc-pass2-ppc64 12.2.1_git20220924-r10 (using abuild 3.11.1-r0)
>>> gcc-pass2-ppc64: Checking sanity of /builds/rebornlinux/aports/main/gcc/APKBUILD...
...
>>> gcc-pass2-ppc64: Updating the main/x86_64 repository index...
>>> gcc-pass2-ppc64: Signing the index...
>>> musl: Building main/musl 1.2.4-r2 (using abuild 3.11.1-r0)
>>> musl: Checking sanity of /builds/rebornlinux/aports/main/musl/APKBUILD...
...
>>> musl: Updating the main/ppc64 repository index...
>>> musl: Signing the index...
>>> gcc-ppc64: Building main/gcc-ppc64 12.2.1_git20220924-r10 (using abuild 3.11.1-r0)
>>> gcc-ppc64: Checking sanity of /builds/rebornlinux/aports/main/gcc/APKBUILD...
...
>>> gcc-ppc64: Updating the main/x86_64 repository index...
>>> gcc-ppc64: Signing the index...
>>> gcc-ppc64: Updating the main/ppc64 repository index...
>>> gcc-ppc64: Signing the index...
>>> build-base-ppc64: Building main/build-base-ppc64 0.5-r3 (using abuild 3.11.1-r0)
...
```

<hr>

### # preliminary analysis
major bootstrap related dir are mounted from gitlab-runner host to job runtime container:  
1 aports: for persistent of the job src code repo, which act as buildroot for alpine linux;  
2 packages: for persistent of the job output apks, organized as packages/${repo}/${arch}/xxx.apk;  
3 logs: for persistent of the job full logs;  
4 apkcached: for persistent of the job apk cache (some apk);  
5 distfiles: for persistent of the job target apk srouce code repo.
```text
$ sudo cat /etc/gitlab-runner/config.toml
[[runners]]
  name = "cloud-server-1"
  url = "https://gitlabe1.ext.net.xxxxx.com/"
  token = "q2SyArbaxwrh_joKBVp7"
  executor = "docker"
  pre_get_sources_script = "git config --global http.proxy $HTTP_PROXY; git config --global https.proxy $HTTPS_PROXY"
  environment = ["https_proxy=http://10.144.1.10:8080", "http_proxy=http://10.144.1.10:8080",
                 "HTTPS_PROXY=http://10.144.1.10:8080", "HTTP_PROXY=http://10.144.1.10:8080"]
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/repo/reborn/sharedhome/aports:/builds/rebornlinux/aports:rw",
               "/repo/reborn/sharedhome/packages:/home/reborn/packages:rw",
               "/repo/reborn/sharedhome/logs:/home/reborn/logs:rw",
               "/repo/reborn/sharedhome/apkcache:/etc/apk/cache:rw",
               "/repo/reborn/sharedhome/distfiles:/var/cache/distfiles:rw", "/cache"]
    shm_size = 0
```

checking the apk up to date determine logic by abuild script src:
judging whether source code & patches & APKBUILD file are newer than apk, if apk is newer, then the apk is up to date.
```text
$ vim $(which abuild)
...
# check if package is up to date
apk_up2date() {
    local i s
    for i in $allpackages; do
        subpkg_set "$i"
        if [ ! -f "$REPODEST/$repo/${subpkgarch/noarch/$CARCH}/$subpkgname-$pkgver-r$pkgrel.apk" ]; then
            subpkg_unset
            return 1
        fi
    done
    subpkg_unset
    [ -n "$keep" ] && return 0

    cd "$startdir"
    # source code, patches, APKBUILD
    for i in $source APKBUILD; do
        if is_remote "$i"; then
            s="$SRCDEST/$(filename_from_uri $i)"
        else
            s="$startdir/${i##*/}"
        fi
        if [ "$s" -nt "$REPODEST/$repo/${pkgarch/noarch/$CARCH}/$pkgname-$pkgver-r$pkgrel.apk" ]; then
            return 1
        fi
    done
    return 0
}

# judging wetheter the generated apks is newer than its corresponding APKINDEX.tar.gz at the same dir,
# if apk file is newer than APKINDEX.tar.gz, then abuild index is not update to date.
abuildindex_up2date() {
    local i

    for i in $allpackages; do
        subpkg_set "$i"
        local dir="$REPODEST"/$repo/${subpkgarch/noarch/$CARCH}
        local idx="$dir"/APKINDEX.tar.gz
        local file="$dir"/$subpkgname-$pkgver-r$pkgrel.apk

        # if any file is missing or .apk is newer then index
        # the index needs to be updated
        if [ ! -f "$idx" -o ! -f "$file" -o "$file" -nt "$idx" ]; then
            subpkg_unset
            return 1
        fi
    done
    subpkg_unset

    return 0
}

up2date() {
    check_arch || return 0
    check_libc || return 0
    apk_up2date && abuildindex_up2date
}
...
```

<hr>

### # root cause
abuildindex_up2date above shall never return 1 in our bootstrap init-ppc64 logic,
by checking the timestamp of aport repo (buildroot), distfiles (source code), apk, found that each time after a new commit to gitlab aport repo,
the job is retriggered, then all mtime of files in self-maintained aport repo is set as the time creating the gitlab ci job.

same issue report can be found at: https://gitlab.com/gitlab-org/gitlab-runner/-/issues/3133

<hr>

### # comparison analysis
we need to found out why gitlab ci/cd job related source code, mounted into gitlab runner host,
after each git fetch, all the timestamp of files will change to job creation time.

given a simple test gitlab ci/cd config file as:
```text
test:
  when: manual
  variables:
    GIT_STRATEGY: clone                           # fetch
  script:
    - stat /builds/rebornlinux/devkit/README.md   # check the inode info
  tags:
    - docker-generic
```

the first attempt using git clone, gitlab ci/cd log as:
```text
Running with gitlab-runner 13.4.0 (4e1f20da) on docker-rebornlinux WG-gzNpK
Preparing the "docker" executor
Using Docker executor with image alpine:latest ...
Pulling docker image alpine:latest ...
Using docker image sha256:f8c2... for alpine:latest with digest alpine@sha256:51b6...
Preparing environment
Running on runner-wg-gznpk-project-64744-concurrent-0 via cloud-server-2...
Getting source from Git repository
Fetching changes with git depth set to 50...
Initialized empty Git repository in /builds/rebornlinux/devkit/.git/
Created fresh repository.
Checking out 3cc34df5 as test...
Skipping Git submodules setup
Executing "step_script" stage of the job script
$ stat /builds/rebornlinux/devkit/README.md
  File: /builds/rebornlinux/devkit/README.md
  Size: 4601      	Blocks: 16         IO Block: 4096   regular file
Device: fc02h/64514d	Inode: 1827213     Links: 1
Access: (0666/-rw-rw-rw-)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2024-01-08 02:21:50.077224077 +0000
Modify: 2024-01-08 02:21:50.077224077 +0000
Change: 2024-01-08 02:21:50.077224077 +0000
Job succeeded
```

the second attempt using git fetch, gitlab ci/cd log as:
```text
Running with gitlab-runner 13.4.0 (4e1f20da) on docker-rebornlinux WG-gzNpK
Preparing the "docker" executor
Using Docker executor with image alpine:latest ...
Pulling docker image alpine:latest ...
Using docker image sha256:f8c2... for alpine:latest with digest alpine@sha256:51b6...
Preparing environment
Running on runner-wg-gznpk-project-64744-concurrent-0 via cloud-server-2...
Getting source from Git repository
Fetching changes with git depth set to 50...
Initialized empty Git repository in /builds/rebornlinux/devkit/.git/
Created fresh repository.
Checking out 10978ab5 as test...
Skipping Git submodules setup
Executing "step_script" stage of the job script
$ stat /builds/rebornlinux/devkit/README.md
  File: /builds/rebornlinux/devkit/README.md
  Size: 4601      	Blocks: 16         IO Block: 4096   regular file
Device: fc02h/64514d	Inode: 1827220     Links: 1
Access: (0666/-rw-rw-rw-)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2024-01-08 02:23:50.612652831 +0000
Modify: 2024-01-08 02:23:50.612652831 +0000
Change: 2024-01-08 02:23:50.612652831 +0000
Job succeeded
```
compare the timestamp printed from the same file of src proj, the second fetch simply fallback to git clone indeed.

let's re-run the gitlab ci/cd job again manually, the log is as:
```text
Running with gitlab-runner 13.4.0 (4e1f20da) on docker-rebornlinux WG-gzNpK
Preparing the "docker" executor
Using Docker executor with image alpine:latest ...
Pulling docker image alpine:latest ...
Using docker image sha256:f8c2... for alpine:latest with digest alpine@sha256:51b6...
Preparing environment
Running on runner-wg-gznpk-project-64744-concurrent-0 via cloud-server-2...
Getting source from Git repository
Fetching changes with git depth set to 50...
Reinitialized existing Git repository in /builds/rebornlinux/devkit/.git/
Checking out 10978ab5 as test...
Skipping Git submodules setup
Executing "step_script" stage of the job script
$ stat /builds/rebornlinux/devkit/README.md
  File: /builds/rebornlinux/devkit/README.md
  Size: 4601      	Blocks: 16         IO Block: 4096   regular file
Device: fc02h/64514d	Inode: 1827220     Links: 1
Access: (0666/-rw-rw-rw-)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2024-01-08 02:23:50.612652831 +0000
Modify: 2024-01-08 02:23:50.612652831 +0000
Change: 2024-01-08 02:23:50.612652831 +0000
Job succeeded
```
we could see that the timestamp of the git fetch job re-run is not changed compared to last git fetch job.
that is to say, between the two jobs, some cache that holds the ci/cd job src proj is re-utilized.

let us change some words in rebornlinux/devkit/README.md, then re-run the job,
after GIT_STRATEGY: fetch, the timestamp related to README is changed,
which conforms to the expected behavior.
```text
Running with gitlab-runner 13.4.0 (4e1f20da) on docker-rebornlinux WG-gzNpK
Preparing the "docker" executor
Using Docker executor with image alpine:latest ...
Pulling docker image alpine:latest ...
Using docker image sha256:f8c2... for alpine:latest with digest alpine@sha256:51b6...
Preparing environment
Running on runner-wg-gznpk-project-64744-concurrent-0 via cloud-server-2...
Getting source from Git repository
Fetching changes with git depth set to 50...
Reinitialized existing Git repository in /builds/rebornlinux/devkit/.git/
Checking out 0b2c6bd4 as test...
Skipping Git submodules setup
Executing "step_script" stage of the job script
$ stat /builds/rebornlinux/devkit/README.md
  File: /builds/rebornlinux/devkit/README.md
  Size: 4615      	Blocks: 16         IO Block: 4096   regular file
Device: fc02h/64514d	Inode: 1827220     Links: 1
Access: (0666/-rw-rw-rw-)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2024-01-08 02:54:50.869532109 +0000
Modify: 2024-01-08 02:54:50.869532109 +0000
Change: 2024-01-08 02:54:50.869532109 +0000
Job succeeded
```

if we maintain self-mounted src proj code and mounted into gitlab runner using above runner config,
each time ci/cd job is doing git clone or git fetch, whatever the relevant files changed or not,
will modify the mtime for the src code tree.

<hr>

### # solutions
◆ method-1 use tag for runner for specific job to re-use the cache, let gitlab cache to manage src code tree & timestamp.

test result: even with named cache for designated dir, the built apks still need to be rebuilt again among different pipeline after each commit.

see https://docs.gitlab.com/ee/ci/caching for details about gitlab ci/cd job caching tricks.  
see blog post gitlab cache for understanding of gitlab shared cache in different level.


◆ method-2 mount src code tree repo into gitlab runner, add a hook function after each default git operation,
before bootstrap stage of ci/cd job.  
the gitlab-ci.yml is like:
```text
.bootstrap_begin: &bootstrap_begin
  - rm -f ~/logs/$CI_JOB_NAME.log; touch ~/logs/$CI_JOB_NAME.log
  - exit_code=0
  - tail -f ~/logs/$CI_JOB_NAME.log | grep -e ">>>" &
  - restore_repo_time_as_last_commit.sh

.bootstrap_end: &bootstrap_end
  - sleep 3
  - test $exit_code -eq 0 || tail -n 400 ~/logs/$CI_JOB_NAME.log
  - exit $exit_code

variables:
  GIT_STRATEGY: fetch

.init:
  image:
    name: rebornlinux-docker-local.artifactory-blr1.int.net.xxxxx.com/rebornlinux/basedev:latest
    entrypoint: [""]
  when: manual
  stage: bootstrap
  timeout: 6 hours

init-aarch64:
  extends: .init
  script:
    - *bootstrap_begin
    - ${CI_PROJECT_DIR}/scripts/bootstrap.sh aarch64 norust,nokernel > ~/logs/$CI_JOB_NAME.log 2>&1 || exit_code=$?
    - *bootstrap_end
  tags:
    - aport-specific
```
the restore_repo_time_as_last_commit.sh could be found at blog post: restore git repo file mtime.
after changing all rebornlinux/aport timestamps, it reported all apk up to date, and the needless rebuilt is skipped.

◆ method-3 discard the gitlab build-in git operation by setting the GIT_STRATEGY as none,
manually clone / fetch the utilized aport repo before each job starts.  
this method can fix the bug, but will not pretain the tag of each gitlab ci/cd pipeline:
```text
$ git log --oneline
fff51afa (HEAD, origin/reborn, refs/pipelines/3089452) bootstrap: debugging
53ca39f0 (refs/pipelines/3089385) gitlab ci: wait before job completes to have full log printed
0fb45da9 (refs/pipelines/3089136) gitlab ci: use basedev as image
86ff2da4 (refs/pipelines/3083468) gitlab-ci/cd: add bootstap stage for buid base
63f60e26 main/libffi: add linux-headers to makedepends for mips
72ad51c7 main/build-base: support cross build dependency on build-base-CTARGET_ARCH
...
```
the tag named pipeline/* is recorded for each pipeline history to trigger job for certain rev of code repo.

<hr>

### # code in action (method #3)
todo
