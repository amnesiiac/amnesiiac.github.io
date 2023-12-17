---
layout: post
title: "shell variables (shell)"
author: "melon"
date: 2023-12-09 11:38
categories: "2023"
tags:
  - shell
---

### # init shell variable using colon
in rebornlinux/devkit/scripts/build.sh, we have the following variable definition:
```text
: "${REPODEST:=$HOME/packages}"
: "${MIRROR:=https://dl-cdn.alpinelinux.org/alpine}"
: "${MAX_ARTIFACT_SIZE:=300000000}" #300M
: "${CI_DEBUG_BUILD:=}"

: "${CI_ALPINE_BUILD_OFFSET:=0}"
: "${CI_ALPINE_BUILD_LIMIT:=9999}"
```
the colon ahead actual means noop in shell, which do nothing, always return 0,
but colon cmd will expand the parameter, so as to assign default value for var inside.
this form of variable assignment is to prevent mis-interpreting of the init value as a cmd to exec (shell safety).

<hr>

### # readonly variable in shell
a code snippet from rebornlinux/devkit/scripts/build.sh to setup basic env var:
```text
readonly APORTSDIR=$CI_PROJECT_DIR                         # ci/cd jobs cloned proj dir
readonly REPOS="main community testing non-free xxxxx"
readonly ARCH=$(apk --print-arch)                          # alpine arch
readonly ARCHS="x86_64 aarch64 mips64 ppc64 ppc"
readonly BASEBRANCH=$CI_MERGE_REQUEST_TARGET_BRANCH_NAME   # from gitlab ci job env
```

a quick test to emphasis the drawbacks when using readonly variable:
```text
# set readonly variables
$ declare -rx logs=hello

# check env variables
$ export
  declare -x DISPLAY="localhost:16.0"
  declare -x USER="metung"
  declare -rx logs="hello"  <--- readonly
  ...

# check env on cur shell
$ echo $logs
  hello

# try set env on cur shell
$ logs=good
  -bash: logs: readonly variable

# try set env on inherited shell
# double quote: before var assigned to subshell, it get expanded, so value in cur shell is echo out
$ bash -c "logs=goodbye; echo $logs"
  hello
# single quote: keep the string pass to subshell static, which will break cur shell readonly env var
$ bash -c 'logs=goodbye; echo $logs'
  goodbye
```
