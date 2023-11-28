---
layout: post
title: shell one-step debug helper (shell)
author: "melon"
date: 2023-09-26 21:54
categories: "2023"
tags:
  - shell
---

### # shell debug helper script 1: press enter to debug step by step 
Using the following script, hit enter to execute shell line by line.  
```text
#!/usr/bin/env bash
                         # only bash support trap debug, if use /bin/sh, wont work
set -x                   # start shell xtrace mode
trap read debug          # ref: https://phoenixnap.com/kb/bash-read

echo "the code line1"
echo "the code line2"
echo "the code line3"
echo "the code line4"
echo "the code line5"
echo "the code line6"
```
But the script has limitations if the script contains code like:  
```text
while read line; do 
    echo $line; 
done < somefile
```
the debug decorator may not work as expected.

By the way, list signals that can be trapped on centos:
```text
$ trap -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```

<hr>

### # shell debug helper script 2: press ctrl+c to debug step by step 
the script below will work even contains "while ... read" clause inside.
```text
#!/usr/bin/env bash

echo "Press CTRL+C to proceed."
trap "pkill -f 'sleep 1h'" INT           # trap INT signal, and execute each bash line with set -x set
trap "set +x ; sleep 1h ; set -x" DEBUG

echo "the code line1"
echo "the code line2"
echo "the code line3"
echo "the code line4"
echo "the code line5"
echo "the code line6"
```

<hr>

### # shell debug helper script 3: sleep 5s between execution of each line
```text
#!/usr/bin/env bash

trap "set +x; sleep 5; set -x" DEBUG

echo "the code line1"
echo "the code line2"
echo "the code line3"
echo "the code line4"
echo "the code line5"
echo "the code line6"
```

<hr>

### # shell debug helper script 3: 
ref: https://selivan.github.io/2022/05/21/bash-debug.html
```text
#!/bin/bash

function _bash_debug_print_usage() {
    echo "Usage: $0 <script to debug> [script args]"
    echo
    echo "bash debugger"
    echo "Each script command is printed before execution"
    echo "Then command prompt appears (no autocompletion)"
    echo "Prompt includes the last command return value"
    echo "You can print a command to execute or an empty line to continue"
    echo "Note: set -T is used, changing DEBUG and RETURN traps inheritance"
}

function _bash_debug_trap_DEBUG() {
    _bash_debug_last_retcode="$?"

    # skip prompt when starting script itself
    # shellcheck disable=SC2016
    if [ "$BASH_COMMAND" == 'source "$_bash_debug_script" "$@"' ]; then
        return
    fi

    echo "# $BASH_COMMAND";
    # -r do not interpret backslash escaped characters
    # -e Use readline in interactive shell
    # -p PROMPT
    while read -r -e -p "($_bash_debug_last_retcode) debug> " _bash_debug_command; do
        if [ -n "$_bash_debug_command" ]; then
            eval "$_bash_debug_command";
        else
            break;
        fi;
    done
}

if [ -z "$1" ]; then
    _bash_debug_print_usage
else
    _bash_debug_script="$1"
    shift
    if [ ! -f "$_bash_debug_script" ]; then
        echo "ERROR: not a readable file: $_bash_debug_script"
        _bash_debug_print_usage
        exit 1
    fi
    set -T                                                      # see man trap & grep debug
    trap '_bash_debug_trap_DEBUG' DEBUG
    # shellcheck disable=SC1090
    source "$_bash_debug_script" "$@"
fi
```
The ussage of "set -T":  
If set, any traps on DEBUG and RETURN are inherited by shell functions, command substitutions, and commands executed in a sub‐shell environment.  The DEBUG and RETURN traps are normally not inherited in such cases.
