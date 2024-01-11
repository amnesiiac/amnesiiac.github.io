---
layout: post
title: "heredoc usages (shell)"
author: "melon"
date: 2023-12-25 21:00
categories: "2023"
tags:
  - shell
---

### # basic usages
using heredoc to input mutiple line into file, the > is line prompt.
```text
$ cat << EOF > file
> [url "https://gitlab-ci-token:@gitlabe1.ext.net.nokia.com"]
>       insteadOf = https://gitlabe1.ext.net.nokia.com
> EOF

$ cat file
[url "https://gitlab-ci-token:@gitlabe1.ext.net.nokia.com"]
      insteadOf = https://gitlabe1.ext.net.nokia.com
```

<hr>

### # code in action: rebornlinux/aports/.gitlab-ci.yml.
the following shows how to use mutltiline heredoc for gitlab yaml script:
```text
default:
  image:
    name: rebornlinux-docker-local.artifactory-blr1.int.net.xxxxx.com/rebornlinux/base:v1.3.1
    entrypoint: [""]
  before_script: |
    cat << EOF > ~/.gitconfig
    [url "https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlabe1.ext.net.xxxxx.com"]
          insteadOf = https://gitlabe1.ext.net.xxxxx.com
    EOF
```

an equivalent form of the above heredoc + yaml usage in gitlab ci/cd config:
```text
default:
  image:
    name: rebornlinux-docker-local.artifactory-blr1.int.net.xxxxx.com/rebornlinux/base:v1.3.1
    entrypoint: [""]
  before_script:
    - git config --global url."https://gitlab-ci-token:${CI_JOB_TOKEN}@xxxxx.com".insteadOf "https://xxxxx.com"
```

<hr>

### # code in action: hostfw/dockerfile/vmm/vm.sh
the following script shows how to use heredoc to print help msg for script:
```text
#!/bin/bash

WORK_DIR="/work"                                                          # default settings

cmd_help() {
    cat <<EOF
A code in action introduction to heredoc usage:

Usage: $(basename $0) [options] command
Options:
    -h, --help   print help
    --workdir    set work directory, /work by default
EOF
}

cmd_test() {
    echo "cmd run from placeholder function with workdir set to $WORK_DIR"
}

main() {
    while [ $# -gt 0 ]; do                                                # parse main line args
        case "$1" in
            -h|--help)
                cmd_help
                exit 0
                ;;
            --workdir)
                WORK_DIR=$2
                test -z "$WORK_DIR" && cmd_help && exit 1
                shift
                ;;
            -*)
                echo "Unknown arg: $1. Please use \`$0 help\` for help."
            ;;
            *)
                break
            ;;
        esac
        shift
    done

    if [ $# = 0 ]; then
        echo "No command provided. Please use \`$0 help\` for help."
        exit 1
    fi

    if ! type "cmd_$1" | grep "function" >/dev/null; then                   # check if $1 is valid to run
        echo "Unknown command: $1. Please use \`$0 help\` for help."
        exit 1
    fi

    cmd=cmd_$1
    shift
    $cmd "$@"                                                               # $@ is now a list of cmd-specific args
}

main "$@"
```

t.sh code use case:
```text
$ ./t.sh --workdir /hello test
this is a cmd run from a placeholder functions, with workdir set to /hello
```
```text
$ ./t.sh --workdir /hello test
this is a cmd run from a placeholder functions, with workdir set to /work
```

print cmd help:
```text
$ ./t.sh -h
A code in action introduction to heredoc usage:

Usage: t.sh [OPTIONS] COMMAND
Options:
    -h|--help   print help
    --workdir   set work directory, /work by default
  cat << EOF
Usage: t.sh [options]

This script demonstrates using a heredoc for help messages.

Options:
  -h, --help     Display this help and exit
  -n NAME        Set name to use
  -a             Display all configuration
```
```text
$ ./t.sh --workdir
A code in action introduction to heredoc usage:

Usage: t.sh [OPTIONS] COMMAND
Options:
    -h|--help   print help
    --workdir   set work directory, /work by default
  cat << EOF
Usage: t.sh [options]

This script demonstrates using a heredoc for help messages.

Options:
  -h, --help     Display this help and exit
  -n NAME        Set name to use
  -a             Display all configuration
```

warnings for invalid or incomplete input:
```text
$ ./t.sh --workdir x
No command provided. Please use `./t.sh help` for help.
```
```text
$ ./t.sh
No command provided. Please use `./t.sh help` for help.
```
