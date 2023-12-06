---
layout: post
title: "nerdctl port mapping across processes on host machine (container)"
author: "melon"
date: 2023-10-07 21:27
categories: "2023"
tags:
  - container
  - ongoing
---

### # spotlight for this implementation
1) filelock: enable cross docker/nerdctl container instance port allocating with no port overlapping.  
2) log: a usefull log api wrapper for multiprocess/thread python project logging.  
3) shell: a simple but easy to use "run shell & derive return val" in python.  
4) parser: a unified parser for both yaml & json by the same API.  
5) port mapping: independant to nerdctl port allocating implementation (dont rely on nerdctl's port allocation). The allocation can cover multiple non-overlapped port allocating in one container instance proc, and non-overlapp port allocating for container instance procs on one host machine.

note: the nerdctl project(<=1.5.0) has bugs when use "-p" to dynamically allocate ports multiple times in the same container instance; and nerctl fails to allocate the right port in multiple container instances.

<hr>

### # file tree of this project
```text
.
├── error.py                            # error code mapping
├── filelock.py                         # self wrapped filelock using fcntl
├── log_ll.py                           # log inner wrapper for builtin logging module
├── log.py                              # outmost log api
├── myrandom.py
├── network.py                          # core port allocator class
├── parser.py
├── shell.py                            # run shell cmd inside python
├── thread_process.py
└── values.yaml                         # test aux file
```

<hr>

### # code
{% raw %}
__network.py__: allocate port across all processses on a host machine, based on the filelock implemented in __filelock.py__. For details see comment inline.
```text
#!/usr/bin/env python3
import os
import re
import stat
import threading
import ipaddress
import socket
import time
import json

import sys
sys.path.append(os.path.dirname(os.path.realpath(__file__)))

import shell as sh
import myrandom as ran
import log as pr
import parser as parser
import filelock as filelock

container_cli = None

def get_container_cli():
    global container_cli
    if container_cli == None:
        _, res = sh.run("docker info 2>/dev/null")
        if res != "" and not "ERROR" in res:
            container_cli = "docker"
        else:
            _, res = sh.run("nerdctl info 2>/dev/null")
            if res != "":
                container_cli = "nerdctl"
            else:
                pr.error("Cannot get container cli! Please check docker or nerdctl deamon!")
                container_cli = None
    return container_cli

class port_manager:
    def __init__(self, owner="host"):
        self.parser = None
        self.owner = owner
        self.lock = filelock.FileLock("/repo/metung/portmanager/host_port_lock")
        self.all_ports = {}
        self.allocator = {}

    def _update_networks(self):
        # get init ports history from host_port.yaml, format as: {"owner1":{port1, port2}, "owner2":{port3,port4}}
        host_port_file = "/repo/metung/portmanager/host_port.yaml"
        if not os.path.exists(host_port_file):
            sh.run("echo {{}} > {} && chmod 777 {}".format(host_port_file, host_port_file))
        self.parser = parser.Parser(host_port_file)
        try:
            self.all_ports = self.parser.load()
        except:
            self.all_ports = {}

        # update known currently used ports in active container (self.all_ports)
        cli = get_container_cli()
        _, text = sh.run("{} ps -a --format '{{{{.Names}}}}'".format(cli))
        all_container_names = text.split("\n")
        for container_name in all_container_names:
            if not container_name:
                continue
            self.all_ports[container_name] = {"expire_timestamp": int(time.time() + 120), "ports": []}
            _, text1 = sh.run("{} port {}".format(cli, container_name))
            container_ports = text1.split("\n")
            for line in container_ports:
                if " -> " in line:
                    port_num = line.split(" -> ")[1].split(":")[1]
                    if port_num != "":
                        self.all_ports[container_name]["ports"].append(int(port_num))

        # clean obsoleted container->port_list pair
        for owner in list(self.all_ports.keys()):
            if owner not in all_container_names:
                now_timestamp = int(time.time())
                if now_timestamp > self.all_ports[owner]["expire_timestamp"]:
                    del self.all_ports[owner]

    def open(self):                                           # acquire lock & update critical area
        self.lock.lock()
        self._update_networks()

    def _available(self, port):                               # check if the port is available
        _, text = sh.run("netstat -lnt")                      # get ports that already listened by other proc on host
        if str(port) not in text:
            for owner in self.all_ports:
                if port in self.all_ports[owner]["ports"]:    # if the port is already allocated by current portmanager
                    return False 
            self.all_ports[self.owner]["ports"].append(port)
            return True
        else:
            return False

    def get(self):                                            # port allocation logic
        base_num = 40000
        allocated_num = base_num
        valid = False
        ret = 0
        range_round = 0
        if self.owner not in self.all_ports:
            self.all_ports[self.owner] = {"expire_timestamp": int(time.time() + 120), "ports": []}
        while True:
            allocated_num = allocated_num + 1
            if ret >= 60000:
                allocated_num = base_num
                range_round += 1
            if range_round >= 2:
                pr.critical("inavailable port in range(40000-60000)")
                break
            if self._available(allocated_num):
                break
        return allocated_num

    def close(self):                                # dump all allocated port into critical area: host_port.yaml
        if self.all_ports:
            self.parser.dump(self.all_ports)
        self.lock.unlock()                          # unlock the filelock

    def destroy(self):
        self.open()
        if self.owner in self.all_ports:
            del self.all_ports[self.owner]
        self.close()

def __get_target_map_port():                                       # apply for a port that unique cross all proc on host
    cli = get_container_cli()
    if cli == "nerdctl":
        pm = port_manager("target_map_{}".format(ran.get_str(6)))  # create port manager obj
        pm.open()                                                  # lock (filelock)
        target_map = pm.get()                                      # get the port
        pm.close()                                                 # unlock 
        return str(target_map)
    else:
        return ""

if __name__ == "__main__":
    port = __get_target_map_port()
    pass
```
__filelock.py__: implementation of a filelock based on __fcntl__ in c.
```text
#!/usr/bin/env python3
import os, stat
import fcntl

class FileLock:
    def __init__(self, filename):
        self.filename = filename                             # file: critical section
        self.locked = False                                  # lock state
        if not os.path.exists(filename):
            file = open(filename, "w")
            file.write("0")
            file.close()
            os.chmod(self.filename, stat.S_IRWXU | stat.S_IRWXG | stat.S_IRWXO)  # set rwx for u/g/o

    def lock(self):
        if self.locked:                                      # check the lock state
            return
        self.file = open(self.filename, "w")                 # open
        fcntl.flock(self.file.fileno(), fcntl.LOCK_EX)       # lock the fd
        self.locked = True
        return self

    def unlock(self):
        if self.locked:                                      # check the lock state
            fcntl.flock(self.file.fileno(), fcntl.LOCK_UN)   # unlock the fd
            self.file.close()
            self.locked = False

    def __enter__(self):
        self.lock()

    def __exit__(self, exc_type, exc_value, traceback):
        self.unlock()
        return None

if __name__ == "__main__":
    import time
    from multiprocessing import Pool
    import myrandom as ran
    from thread_process import *

    def withlock(n):
        d = ran.get_int(3) / 1000
        with FileLock("/repo/metung/portmanager/locktest"):  # the class FileLock has no __enter__ -> error
            print("+[{}]".format(n))
            time.sleep(d)
            print("-[{}]".format(n))
            time.sleep(d)

    def nolock(n):
        d = ran.get_int(3) / 1000
        print("+[{}]".format(n))
        time.sleep(d)
        print("-[{}]".format(n))
        time.sleep(d)

    def t(n):
        times = 0
        while times < 10:
            withlock(n)
            # nolock(n)
            times += 1

    p1 = MyProcess(t, 1)
    p2 = MyProcess(t, 2)
    p3 = MyProcess(t, 3)

    time.sleep(10)

    p1.terminate()
    p2.terminate()
    p3.terminate()
```
test the filelock implementation using self-wrapped process implemented in __thread_process.py__.  
The result shows that the operations inside "with block" are guaranteed to be "atomic" & "uninterruptable" between processes.
```text
# $ python3 filelock.py withlock
# +[1]
# -[1]
# +[2]
# -[2]
# +[3]
# -[3]
# +[1]
# -[1]
# +[2]
# -[2]
...
```
```text
# $ python3 filelock.py  # nolock
# +[1]
# +[2]
# +[3]
# -[3]
# +[3]
# -[1]
# -[2]
# +[1]
# -[1]
# -[3]
# +[1]
...
```
__shell.py__: run shell command and derive result & code in python.
```text
import subprocess
import os, sys, traceback

sys.path.append(os.path.dirname(os.path.realpath(__file__)))
import signal
import time
from log_ll import pr
from error import InternalError

# run shell cmd & return in python
def run(cmd, wait_timeout=None, output=False, log_path=None, no_stderr=True, silent=False):
    return_code = 0
    return_text = ""
    return_error = ""

    if log_path != None:
        cmd = "{} |& tee {}".format(cmd, log_path)

    # umask is supported since python3.9, we should call it explicitly to make created files removable by users
    cmd = "umask 0 && " + cmd
    process = subprocess.Popen(
        cmd,
        stdin=subprocess.DEVNULL,  # TODO:also prompt sudo password. if timeout, terminal is abnormal.
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
        pr.warning("SystemExit(%s) occurs during cmd : %s", e, cmd)
        sys.exit(e)
    except subprocess.TimeoutExpired as e:
        process.kill()
        process.returncode = InternalError.err_timeout
    except:
        pr.warning("Exception: %s\n%s", sys.exc_info(), traceback.format_exc())
        process.kill()
        return_text, return_error = process.communicate()
    return_code = process.returncode
    return_text = return_text.strip()

    # silent mode: disable both the terminal output and the log file output
    if not silent:
        # output: show log on terminal or note down to log file
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
    (status, text) = run("ping 135.251.200.81", 2, True)
    print(status)
    print(text)
```
__log.py__: the outside log wrapper to enable multiple level log output including: info, warning, debug, error, critical.
```text
import os, sys, io
import traceback
from log_ll import *
from error import error
from error import Error

# pr.error prints call stack, 
# pr.critical prints call stack and exit, be CAUTIOUS to use pr.critical in lower-level modules!

class MylogWrapper(Mylog):
    def set_args(self, log_file, banner=None):
        error.set_logdir(os.path.dirname(os.path.relpath(log_file)))
        super().set_args(log_file, banner)

    def error(self, msg, *args, errno=-1):
        errmsg = msg % args
        callstack = r"{}".format("".join(traceback.format_stack()[:-1]))
        error.set_error(errno, errmsg, callstack)
        self.logger.error(errmsg)
        self.logger.error("call stack:\n" + callstack)

    def critical(self, msg, *args, errno=-1):
        errmsg = msg % args
        callstack = r"{}".format("".join(traceback.format_stack()[:-1]))
        error.set_error(errno, errmsg, callstack)
        self.logger.critical(errmsg)
        self.logger.critical("call stack:\n" + callstack)
        sys.exit(errno)

pr = MylogWrapper()

if __name__ == "__main__":
    pr.set_args("./all")
    pr.info("test %d", 1)
    pr.info(r"%d")
    cur = error.get_subdir_errors_and_evaluate_current("../../test/workdir/host/")
    if cur:
        # convert to json file for clicktest
        json_file = os.path.splitext(error.get_logfile())[0] + ".json"
        import utils.parser as ps

        p = ps.Parser(json_file)
        p.dump(cur[0])
    pr.error("test %d", 1, errno=-2)
    pr.error("download failed!", errno=Error.err_artifactory_download)
    a = 111
    b = (1, 2, 3)
    c = {"a": 1}
    pr.info("111", a)
    pr.info("111", b)
    pr.info("111", c)
    pr.info(a)
    pr.info(b)
    pr.info(c)
    pr.debug("hello , %d", a)
    pr.info("hello, %d", 2)
    pr.warning("hello %d" % 3)
    pr.error("hello %s" % "error")
    pr.critical("hello, %s", "critical")
```
__log_ll.py__: the inner implementation for outside log.py wrapper, using python module __logging__ & __traceback__ as a base.
```text
import os, sys, io
import logging
import traceback

console_handler = None

class Mylog:
    DEBUG = logging.DEBUG
    INFO = logging.INFO
    WARNING = logging.WARNING
    ERROR = logging.ERROR

    def __init__(self, level=logging.DEBUG):
        global console_handler
        self.is_set_args = False
        self.log_file = ""
        self.logger = logging.getLogger()
        self.logger.setLevel(level)

        if not console_handler:
            console_handler = logging.StreamHandler()
            self.console_handler = console_handler
            self.console_handler.setLevel(logging.INFO)
            self.console_formatter = logging.Formatter("[%(asctime)s %(levelname)8s]: %(message)s", "%H:%M:%S")
            self.console_handler.setFormatter(self.console_formatter)
            self.logger.addHandler(self.console_handler)

    def set_args(self, log_file, banner=None):
        if not log_file:
            return
        if self.is_set_args:
            return
        self.is_set_args = True
        log_file = os.path.realpath(log_file)
        self.log_file = log_file

        self.file_handler = logging.FileHandler(filename=log_file)
        self.file_handler.setLevel(logging.DEBUG)
        self.file_formatter = logging.Formatter("[%(asctime)s %(threadName)s %(levelname)8s]: %(message)s", 
            "%Y%m%d %H:%M:%S")
        self.file_handler.setFormatter(self.file_formatter)
        self.logger.addHandler(self.file_handler)
        if banner:
            with open(self.log_file, "w") as f:
                f.write(banner)
        self.logger.info("please see log in {}".format(log_file))

    def get_log_file(self):
        return self.log_file

    def sprintf(self, *args):
        sio = io.StringIO()
        print(*args, file=sio)
        return sio.getvalue()

    def debug(self, msg, *args):
        self.logger.debug(msg, *args)

    def info(self, msg, *args):
        self.logger.info(msg, *args)
        # self.logger.info(self.sprintf(*args))

    def warning(self, msg, *args):
        self.logger.warning(msg, *args)

    def error(self, msg, *args, errno=-1):
        self.logger.error(msg, *args)

    def critical(self, msg, *args, errno=-1):
        self.logger.critical(msg, *args)

    def breakpoint(self, dict, title="", level="warning"):
        if level == "warning":
            self.logger.warning(">" * 80)
            if title:
                self.logger.warning("[{}]".format(title))
            for key, value in dict.items():
                self.logger.warning("  %s: %s", key, value)
            self.logger.warning("<" * 80)
        elif level == "debug":
            self.logger.debug(">" * 80)
            if title:
                self.logger.debug("[{}]".format(title))
            for key, value in dict.items():
                self.logger.debug("  %s: %s", key, value)
            self.logger.debug("<" * 80)

pr = Mylog()

if __name__ == "__main__":
    pr.set_args("./all")
    pr.info("test %d", 1)
    pr.info(r"%d")
    a = 111
    b = (1, 2, 3)
    c = {"a": 1}
    pr.info("111", a)
    pr.info("111", b)
    pr.info("111", c)
    pr.info(a)
    pr.info(b)
    pr.info(c)
    pr.debug("hello , %d", a)
    pr.info("hello, %d", 2)
    pr.warning("hello %d" % 3)
    pr.error("hello %s" % "error")
    pr.critical("hello, %s", "critical")
```
__thread_process.py__: a wrapper file of python threading & multiprocessing module.
```text
import os
import signal
import threading
import multiprocessing
from log import pr

def thread_run(func, *para, daemon=True):
    t = threading.Thread(target=func, args=para, daemon=daemon)
    t.start()
    return t

class MyThread(threading.Thread):
    def __init__(self, func, *para, daemon=True, name="testthread"):
        self.user_func = func
        self.user_para = para
        threading.Thread.__init__(self, name=name, daemon=daemon)
        threading.Thread.start(self)

    def run(self):
        self.user_func(*self.user_para)

class MyProcess:
    def __init__(self, func, *args):
        self.p = multiprocessing.Process(target=func, args=args)
        self.p.start()

    def terminate(self):
        self.p.terminate()

if __name__ == "__main__":
    pass
```
__error.py__: maintaining error categories mapping, and provide API for error obj handling & persistency (to log file):
```text
import os
import parser as ps
import threading
import copy
from log_ll import pr

lock = threading.Lock()

# create error obj & save it to corresponding dirs:
# {
#     current:[
#         {errno : x
#         errno_strings : [ERR_CATEGORY, ERR_STR]
#         errmsg : "XXX"
#         callstack : "XXXX"
#         source: SUB-SUBDIR1 }, #existing if it comes from subdir
#         
#         {errno:x, errno_strings:[x,x], errmsg:xxx, callstack:xxxx}  #another error
#         ...
#     ]
# 
#     SUBDIR1 : {
#         current: []
#         SUB-SUBDIR1: {...}
#     }
# }

class InternalError:
    success = 0
    err_timeout = -100

class Error:
    def __init__(self):
        self.errno = Error.err_success
        self.errstrings = Error.err_to_strings(self.errno)
        self.errmsg = "success"
        self.callstack = ""
        self.logfile = ""

    err_category_env = 1 << 24
    err_category_hostfw = 2 << 24
    err_category_board = 3 << 24
    err_category_testfw = 4 << 24
    err_category_testcases = 5 << 24
    err_category_input = 6 << 24
    err_category_build = 7 << 24

    err_success = 0

    err_artifactory = err_category_env + 1
    err_search_file = err_category_env + 2
    err_infra_issue = err_category_env + 3
    err_download_dockerimage = err_category_env + 4
    err_no_disk_quota = err_category_env + 5
    err_timeout_without_board_failure = err_category_env + 6

    err_hostfw_invalid_param = err_category_hostfw + 1
    err_hostfw_general = err_category_hostfw + 2

    err_artifactory_download = err_category_build + 1
    err_local_download = err_category_build + 2
    err_daily_download = err_category_build + 3

    err_board_start_failure = err_category_board + 1
    err_board_cfg_error = err_category_board + 2
    err_board_image_error = err_category_board + 3
    err_board_start_timeout = err_category_board + 4

    err_testfw_cfg_error = err_category_testfw + 1
    # external pass invalid param in, typically testfw(clicktest, batch)
    err_invalid_param = err_category_testfw + 2

    err_test_cases_failure = err_category_testcases + 1

    @staticmethod
    def err_category_to_string(cat):
        ret = "UNKNOWN"
        if cat == Error.err_category_env:
            ret = "ENV FAILURE"
        elif cat == Error.err_category_hostfw:
            ret = "HOSTFW FAILURE"
        elif cat == Error.err_category_board:
            ret = "BOARD FAILURE"
        elif cat == Error.err_category_testfw:
            ret = "BATCH FAILURE"
        elif cat == Error.err_category_testcases:
            ret = "TI FAILURE"
        elif cat == Error.err_category_input:
            ret = "INPUT PARAMS FAILURE"
        elif cat == Error.err_category_build:
            ret = "HOST BUILD INVALID"
        return ret

    @staticmethod
    def err_to_strings(err):
        cat = Error.err_category_to_string(err & 0xFF000000)
        ret = "UNKNOWN"
        if err == Error.err_success:
            ret = "SUCCESS"
        elif err == Error.err_download_dockerimage:
            ret = "DOWNLOAD DOCKERIMAGE FAILED"
        elif err == Error.err_artifactory:
            ret = "ARTIFACTORY NOT ACCESSIBLE"
        elif err == Error.err_search_file:
            ret = "SEARCH FILE FAILED"
        elif err == Error.err_infra_issue:
            ret = "INFRASTRUCTURE_ISSUE"
        elif err == Error.err_hostfw_invalid_param:
            ret = "HOSTFW INVALID PARAMS"
        elif err == Error.err_artifactory_download:
            ret = "INVALID ARTIFACTORY URL/BLDNUM/RELEASE BUILD PARAM"
        elif err == Error.err_daily_download:
            ret = "INVALID DAILY URL BUILD PARAM"
        elif err == Error.err_local_download:
            ret = "INVALID LOCAL PATH BUILD PARAM"
        elif err == Error.err_hostfw_general:
            ret = "GENERAL HOSTFW ERROR"
        elif err == Error.err_board_start_failure:
            ret = "BOARD START FAILURE"
        elif err == Error.err_board_start_timeout:
            ret = "BOARD START TIMEOUT"
        elif err == Error.err_board_cfg_error:
            ret = "BOARD CONFIGURATION ERROR"
        elif err == Error.err_board_image_error:
            ret = "BOARD IMAGE ERROR"
        elif err == Error.err_testfw_cfg_error:
            ret = "TESTFW CONFIG ERROR"
        elif err == Error.err_invalid_param:
            ret = "TESTFW PASS INVALID PARAM"
        elif err == Error.err_test_cases_failure:
            ret = "TEST CASES FAILURE"
        elif err == Error.err_timeout_without_board_failure:
            ret = "TIMEOUT WITHOUT BOARD FAILURE"

        return [cat, ret]

    def set_logdir(self, dir):
        with lock:
            if not self.logfile:
                self.logfile = os.path.join(os.path.realpath(dir), "error.yaml")
                p = ps.Parser(self.logfile)

    def get_logfile(self):
        with lock:
            return self.logfile

    def set_error(self, errno, errmsg, callstack):
        self.errno = errno
        self.errstrings = Error.err_to_strings(errno)
        self.errmsg = errmsg
        self.callstack = callstack
        if self.logfile and errno != Error.err_success:
            with lock:
                p = ps.Parser(self.logfile)
                cur = p.get("current")
                if cur is None:
                    cur = [{"errno": errno, 
                            "errno_strings": self.errstrings, 
                            "errmsg": errmsg, "callstack": callstack}]
                    json_file = os.path.splitext(self.logfile)[0] + ".json"
                    j = ps.Parser(json_file)
                    j.dump(cur[0])
                else:
                    cur += [{"errno": errno, 
                             "errno_strings": self.errstrings, 
                             "errmsg": errmsg, "callstack": callstack}]
                p.set("current", cur)
                p.dump()

    def get_last_errorno(self):
        return self.errno

    def get_subdir_errors(self, dir):
        dir = os.path.realpath(dir)
        logfile = os.path.join(dir, "error.yaml")
        d = None
        if os.path.isfile(logfile):
            pr.info("get_subdir_errors : %s", logfile)
            p = ps.Parser(logfile)
            d = p.load()
            if self.logfile and d:
                with lock:
                    p = ps.Parser(self.logfile)
                    p.set(dir, d)
                    p.dump()
            else:
                pr.warning("get_subdir_errors: sub_errors=%s, self.logfile=%s", d, self.logfile)
        else:
            pr.debug("get_subdir_errors : %s not existing.", logfile)
        return d

    # insert first error of subdirs to current errors
    def evaluate_current_errors(self):
        cur = []
        if self.logfile:
            with lock:
                p = ps.Parser(self.logfile)
                d = p.load()
                keys = list(d.keys())
                if len(keys) == 0:
                    return cur

                # get the first error of all subdirs
                cur = p.get("current", [])
                for k in keys:
                    if k != "current":
                        cur.insert(0, copy.deepcopy(d[k]["current"][0]))
                        cur[0]["source"] = k
                        p.set("current", cur)
                        p.dump()
                        break
        return cur

    def get_current_errors(self):
        cur = None
        if self.logfile:
            with lock:
                p = ps.Parser(self.logfile)
                cur = p.get("current")
        return cur

    def get_subdir_errors_and_evaluate_current(self, dir):
        self.get_subdir_errors(dir)
        return self.evaluate_current_errors()

error = Error()

if __name__ == "__main__":
    error.set_logdir("./workdir/log/")
    error.get_subdir_errors_and_evaluate_current("./workdir/log/app")
```
__parser.py__: an unified parser with yaml, json format supported, based on the python module yaml json.
```text
import os
import yaml
import json
from log_ll import pr

# TODO: import pr, and avoid cycle dependency

class ParserYaml:
    def __init__(self, f):
        self.f = f
        self.d = {}

    def load(self, obj=None):
        if obj is None:
            if os.path.isfile(self.f):
                with open(self.f) as f:
                    self.d = yaml.load(f, Loader=yaml.FullLoader)
        else:
            self.d = yaml.load(obj, Loader=yaml.FullLoader)
        return self.d

    def dump(self, obj=None, out_f=None, default_flow_style=False):
        if not out_f:
            out_f = self.f
        if obj is None:
            obj = self.d
        if os.path.isdir(os.path.dirname(out_f)):
            try:
                with open(out_f, "w") as f:
                    f.write(yaml.dump(obj, default_flow_style=default_flow_style, sort_keys=False))
                    return obj
            except OSError as err:  # specially for OSError: quota exceed
                pr.critical(err, errno=Error.err_category_env)
        else:
            pr.warning("Parser dump out file %s path not existing", out_f)
            return None

class ParserJson:
    def __init__(self, f):
        self.f = f
        self.d = {}

    def load(self, obj=None):
        if os.path.isfile(self.f):
            with open(self.f) as f:
                self.d = json.loads(f.read())
        return self.d

    def dump(self, obj=None, out_f=None, default_flow_style=False):
        if not out_f:
            out_f = self.f
        if obj is None:
            obj = self.d
        try:
            with open(out_f, "w") as f:
                f.write(json.dumps(obj))
        except OSError as err:  # specially for OSError: quota exceed
            pr.critical(err, errno=Error.err_category_env)

class JsonObj:
    def pretty(self, intput):
        ret = json.dumps(intput, indent=4, separators=(",", ": "))
        return ret

jsonObj = JsonObj()

class YamlObj:
    @staticmethod
    def get(path, base):
        if not path:
            return base

        subs = path.split(".")
        for i, s in enumerate(subs):
            if not s:
                continue
            if s.startswith("[") and i == 0 and s[1:-1].isdigit():
                index = int(s[1:-1])
                if not isinstance(base, list) or index >= len(base):
                    return None
                base = base[index]
            elif s.endswith("]") and not s.startswith("["):
                fields = s.split("[")
                name = fields[0]
                if fields[1][:-1].isdigit():
                    index = int(fields[1][:-1])
                    if not isinstance(base, dict):
                        return None
                    base = base.get(name)
                    if not isinstance(base, list) or index > len(base):
                        return None
                    base = base[index]
                else:
                    return None
            elif isinstance(base, dict):
                base = base.get(s, None)
            else:
                return None

            if base is None:
                break
        return base

    @staticmethod
    def set(path, value, base):
        # TODO: could be recursive, otherwise only support setting values whose parent is existing
        if not path:
            return base

        subs = path.split(".")
        if subs[-1].endswith("]"):
            fields = subs[-1].split("[")
            name = fields[0]
            if fields[1][:-1].isdigit():
                index = int(fields[1][:-1])
                parent = ".".join(subs[0:-1] + [name])
                key = int(index)
            else:
                return None
        elif len(subs) == 1:
            parent = None
            key = subs[-1]
        else:
            parent = ".".join(subs[0:-1])
            key = subs[-1]

        obj = YamlObj.get(parent, base)
        if(isinstance(obj, list) and isinstance(key, int) and key < len(obj)) or isinstance(obj, dict):
            obj[key] = value
            return obj
        else:
            pr.warning("yamlobj.set invalid path %s", path)
            return None

class Parser:
    def __init__(self, f):
        self.root = None
        ext = os.path.splitext(f)[-1]
        if ext in [".yaml", ".yml"]:
            self.p = ParserYaml(f)
        elif ext == ".json":
            self.p = ParserJson(f)
        else:
            raise ValueError(
                "\nThe configuration file format only supports '.yaml', '.yml' or '.json'"
                "\nplease check your file: {}".format(f)
            )

    def load(self, obj=None):
        self.root = self.p.load(obj)
        return self.root

    def get_root(self):
        return self.root

    def get(self, path, base=None):
        if self.root is None:
            self.load()
        if not base:
            base = self.root
        return YamlObj.get(path, base)

    def set(self, path, value, base=None):
        if self.root is None:
            self.load()
        if not base:
            base = self.root
        return YamlObj.set(path, value, base)

    def dump(self, obj=None, out_f=None, default_flow_style=False):
        return self.p.dump(obj, out_f, default_flow_style)

if __name__ == "__main__":
    p = Parser("./values.yaml")
    data = p.load()
    obj = p.get("melons[1].origin")
    import pdb; pdb.set_trace()
    print(obj)
```
__value.yaml__: used in above __parser.py__ module test is as follows:
```text
melons:
  - origin: xinjiang
    block: 1
    slot: 1
  - origin: chifeng
    block: 1
    slot: 2
bananas:
  - origin: xinjiang
    block: 2
    slot: 5
  - origin: chifeng
    block: 3
    slot: 6
```
__myrandom.py__: a random wrapper for python random module, provide random str & random int.
```text
import random
import string

def get_str(len=8, scope=""):
    if not scope or not isinstance(scope, str):
        ret = "".join(random.SystemRandom().choice(string.ascii_lowercase + string.digits) for _ in range(len))
    else:
        ret = "".join(random.SystemRandom().choice(scope) for _ in range(len))
    return ret

def get_int(len=5, fro=None, to=None):
    if fro == None or to == None:
        ret = "".join(random.SystemRandom().choice(string.digits) for _ in range(len))
    else:
        ret = random.randint(fro, to)
    return int(ret)

if __name__ == "__main__":
    s = get_str(8, "0123456789ABCDEF")
    print(s)
```
{% endraw %}
