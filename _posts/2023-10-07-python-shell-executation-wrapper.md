---
layout: post
title: "execute shell cmd in python runtime (python, shell)"
author: "melon"
date: 2023-10-07 21:27
categories: "2023"
tags:
  - python
---

the following is a workable code for it:

```text
import subprocess
import os, sys, traceback

import signal
import time

# try include hostfw project utilities
try:
    sys.path.append(os.path.join(os.path.dirname(os.path.realpath(__file__)), "../../pkg_host"))
    from log_ll import pr
    from error import InternalError
except ModuleNotFoundError:
    standalone = True  # test this script not in the hostfw project context

# run shell cmd & return in python
def run(cmd, wait_timeout=None, output=False, log_path=None, no_stderr=True, silent=False):
    print("-"*48)
    msg = f"run cmd: {cmd} \nwith timeout set as: {wait_timeout}"
    print(msg)
    print("-"*48)

    return_code = 0
    return_text = ""
    return_error = ""

    if log_path != None:
        cmd = "{} |& tee {}".format(cmd, log_path)

    # umask in preexec_fn is supported since python3.9, instead the alternative solution is:
    # umask is set to make file created by this process can be operated by users
    cmd = "umask 0 && " + cmd
    process = subprocess.Popen(
        cmd,
        stdin=subprocess.DEVNULL,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE if no_stderr else subprocess.STDOUT,
        universal_newlines=True,
        # preexec_fn=initchildproc,
        shell=True,
        executable="/bin/bash",
        errors="ignore",
    )

    try:
        return_text, return_error = process.communicate(timeout=wait_timeout)
    except SystemExit as e:
        if standalone:
            print(f"Systemexit({e}) occurs during cmd: {cmd}")
        else:
            pr.warning("SystemExit(%s) occurs during cmd : %s", e, cmd)
        sys.exit(e)
    except subprocess.TimeoutExpired as e:
        process.kill()
        process.returncode = InternalError.err_timeout if not standalone else "timeouterror"
    except:
        if standalone:
            print(f"Exception: {sys.exc_info()}\n{traceback.format_exc()}")
        else:
            pr.warning("Exception: %s\n%s", sys.exc_info(), traceback.format_exc())
        process.kill()
        return_text, return_error = process.communicate()

    return_code = process.returncode
    return_text = return_text.strip()

    if not standalone:
        # silent mode: disable both the terminal output and the log file output
        if not silent:
            # output=true: print log on tty & save debug log to disk
            (pr.info if output else pr.debug)("run cmd: %s", cmd)
            (pr.info if output else pr.debug)("run cmd wait_timeout: %s", wait_timeout)
            if return_text:
                (pr.info if output else pr.debug)("--->")
                (pr.info if output else pr.debug)("cmd output: \n%s", return_text)
                (pr.info if output else pr.debug)("<---")
            if return_error:
                (pr.info if output else pr.debug)("--->")
                (pr.info if output else pr.debug)("cmd error: \n%s", return_error)
                (pr.info if output else pr.debug)("<---")
            (pr.info if output else pr.debug)("cmd return %d", return_code)

    return (return_code, return_text)

if __name__ == "__main__":
    (status, text) = run("ls -la", None, True)
    print(status)
    print(text)
    (status, text) = run("ping 135.251.200.81", 2, True)
    print(status)
    print(text)
```


program test output:

```text
$ python3 t.py
------------------------------------------------
run cmd: ls -la
with timeout set as: None
------------------------------------------------
0
total 2952
drwxr-xr-x   14 mac  staff      448 Jul  6 19:26 .
drwxr-xr-x+ 140 mac  staff     4480 Jul  6 18:58 ..
-rw-r--r--@   1 mac  staff     6148 May 27 22:20 .DS_Store
-rw-r--r--    1 mac  staff     6484 Jun  2 16:57 .clang-format
-rw-r--r--    1 mac  staff      392 Jul 26  2023 Makefile
drwxr-xr-x   18 mac  staff      576 Jun  2 15:49 amnesia
drwxr-xr-x    4 mac  staff      128 Oct  5  2023 apue
drwxr-xr-x@   5 mac  staff      160 Jun  5 21:30 c
drwxr-xr-x    7 mac  staff      224 Jun  1 10:18 cpp
-rw-r--r--    1 mac  staff      815 Jun  2 14:57 hello.cpp
drwxr-xr-x   27 mac  staff      864 Jun  6 21:52 llm
-rw-r--r--    1 mac  staff     3261 Jul  6 19:26 t.py
drwxr-xr-x   24 mac  staff      768 May 22 22:46 udrv
------------------------------------------------
run cmd: ping 135.251.200.81
with timeout set as: 2
------------------------------------------------
timeouterror
```

<hr>

### # subprocess.Popen().communicate() vs subprocess.Popen().wait()
this section answers the question: why choose comminicate method for the shell execution wrapper.

1 communicate() is suitable to be used with stdout=PIPE & stderr=PIPE.
it will block the parent until the parent process terminate and return the stdout & stderr back to parent.

<p style="margin-bottom: 20px;"></p>

2 wait() will deadlock when using stdout=PIPE or stderr=PIPE.
if the child process generates
enough output to the stdout/stderr pipe and suck that the child blocks waiting for the os pipe buffer
to accept more (if parent continously read from the other end).
however, the parent actually consume nothing but busy waiting for the child's termination.
