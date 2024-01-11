---
layout: post
title: "shell variables (shell)"
author: "melon"
date: 2023-12-09 11:38
categories: "2023"
tags:
  - shell
---

### # shell variable basic usage
shell variable initialization
```text
${parameter:-defaultValue}      get default shell variables value
${parameter:=defaultValue}      set default shell variables value
${parameter:?"Error Message"}   display an error message if parameter is not set
```

the colon usage explanation:
```text
${parameter:-defaultValue}      if parameter is not present or unset, return defaultValue to caller.
${parameter-defaultValue}       if parameter is no present, return defaultValue.
```
the colon typically do nothing but expand the parameter, which is used to prevent
execution of defaultValue as a shell cmd.

other usages
```text
${#var}	                        find the length of the string

${var%pattern}                  remove from shortest rear (end) pattern
${var%%pattern}	                remove from longest rear (end) pattern
${var:num1:num2}                substring

${var#pattern}                  remove from shortest front pattern
${var##pattern}	                remove from longest front pattern

${var/pattern/string}	        find and replace (only replace first occurrence)
${var//pattern/string}	        find and replace all occurrences

${!prefix*}                     expands to the names of variables whose names begin with prefix.

${var,} ${var,pattern}	        convert first character to lowercase.
${var,,} ${var,,pattern}        convert all characters to lowercase.
${var^} ${var^pattern}	        convert first character to uppercase.
${var^^} ${var^^pattern}        convert all character to uppercase.
```

<hr>

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
$ declare -rx logs=hello                        # set readonly variables

$ export                                        # check env variables
  declare -x DISPLAY="localhost:16.0"
  declare -x USER="metung"
  declare -rx logs="hello"                      # readonly
  ...

$ echo $logs                                    # check env on cur shell
  hello

$ logs=good                                     # try set env on cur shell
  -bash: logs: readonly variable
```

```text
$ bash -c "logs=goodbye; echo $logs"            # try set env on inherited shell
  hello
```
analysis: before var assigned to subshell, double quoted var get expanded, so value in cur shell is echo out.

```text
$ bash -c 'logs=goodbye; echo $logs'            # try set env on inherited shell
  goodbye
```
analysis: single quote keep the string pass to subshell static, which will break cur shell readonly env var
