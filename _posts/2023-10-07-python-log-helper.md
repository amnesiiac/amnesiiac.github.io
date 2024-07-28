---
layout: post
title: "logging module (python, logging)"
author: "melon"
date: 2023-10-07 21:27
categories: "2023"
tags:
  - python
---

todo: make a self-contained logging wrapper implemention with test result here!!!

a logging helper module implementated in python, used as the log engine in multi-processing &
multi-threading project.
the features for this logging wrapper including:

1) logging banner support.  
2) logging level support: info, warning, debug, error, critical\...  
3) log to the tty & redirection to workdir according to the log level.  
4) log to workdir & subdirs accoriding to the dir setup accordingly.

<hr>

### # logging helper code in action
the toy logging module dir organization:

```text
.
├── error.py        # project specific error code definition (not like linux generall err)
├── log.py          # outmost wrapper for log_ll
├── log_ll.py       # low-level wrapper for python logging & traceback module
└── parser.py       # yaml/json parser helper
```

<p style="margin-bottom: 20px;"></p>

1 log.py: outside wrapper for log_ll.py:

```text
import os, sys, io
import traceback

sys.path.append(os.path.dirname(os.path.realpath(__file__)))  # add current script path to $PATH
from log_ll import *
from error import error, Error

# pr.error prints call stack, pr.critical prints call stack and exit,
# cautious to use pr.critical in lower-level modules!

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

<p style="margin-bottom: 20px;"></p>

2 log_ll.py: the low-level stuff for log.py, wrapper class for python module logging & traceback.

```text
import os, sys, io
import logging
import traceback

"""
low level log module which doesn't depend on external modules, for some utils modules to avoid cycle dependencies.
"""

console_handler = None


class ColoredFormatter(logging.Formatter):
    COLORS = {
        "ERROR": "\033[91m",  # RED
        "CRITICAL": "\033[91m",  # RED
        "WARNING": "\033[95m",  # PURPLE
        "RESET": "\033[0m",  # NULL
    }

    def format(self, record):
        log_message = super().format(record)
        log_level_color = self.COLORS.get(record.levelname, "")
        return f"{log_level_color}{log_message}{self.COLORS['RESET']}"


class Mylog:
    DEBUG = logging.DEBUG
    INFO = logging.INFO
    WARNING = logging.WARNING
    ERROR = logging.ERROR

    def __init__(self, level=logging.DEBUG):
        global console_handler
        self.is_set_args = False
        self.log_file = ""
        # use the default logger, otherwise the artifactory module will add a new handler,
        # and we will meet the duplicate output issue.
        self.logger = logging.getLogger()
        self.logger.setLevel(level)

        if not console_handler:
            console_handler = logging.StreamHandler()
            self.console_handler = console_handler
            self.console_handler.setLevel(logging.INFO)
            self.console_formatter = ColoredFormatter("[%(asctime)s %(levelname)8s]: %(message)s", "%H:%M:%S")
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
        self.file_formatter = logging.Formatter(
            "[%(asctime)s %(threadName)s %(levelname)8s]: %(message)s", "%Y%m%d %H:%M:%S"
        )
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

<p style="margin-bottom: 20px;"></p>

3 error.py: maintain error categories mapping, and provide api for error obj creation & persistency.

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

<p style="margin-bottom: 20px;"></p>

4 parser.py: code can be pasted from blog post python-yaml-json-parser.

```text
left blank deliberately
```

<p style="margin-bottom: 20px;"></p>

5 test the outmost logging interface functionalities, the result is as:

```text
$ python3 log.py
[22:15:48     INFO]: please see log in /Users/mac/code/all
[22:15:48     INFO]: test 1
[22:15:48     INFO]: %d
[22:15:48    ERROR]: test 1
[22:15:48    ERROR]: call stack:
  File "/Users/mac/code/log.py", line 44, in <module>
    pr.error("test %d", 1, errno=-2)

[22:15:48    ERROR]: download failed!
[22:15:48    ERROR]: call stack:
  File "/Users/mac/code/log.py", line 45, in <module>
    pr.error("download failed!", errno=Error.err_artifactory_download)

--- Logging error ---
Traceback (most recent call last):
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 1100, in emit
    msg = self.format(record)
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 943, in format
    return fmt.format(record)
  File "/Users/mac/code/log_ll.py", line 21, in format
    log_message = super().format(record)
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 678, in format
    record.message = record.getMessage()
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 368, in getMessage
    msg = msg % self.args
TypeError: not all arguments converted during string formatting
Call stack:
  File "/Users/mac/code/log.py", line 49, in <module>
    pr.info("111", a)
  File "/Users/mac/code/log_ll.py", line 80, in info
    self.logger.info(msg, *args)
Message: '111'
Arguments: (111,)
--- Logging error ---
Traceback (most recent call last):
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 1100, in emit
    msg = self.format(record)
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 943, in format
    return fmt.format(record)
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 678, in format
    record.message = record.getMessage()
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 368, in getMessage
    msg = msg % self.args
TypeError: not all arguments converted during string formatting
Call stack:
  File "/Users/mac/code/log.py", line 49, in <module>
    pr.info("111", a)
  File "/Users/mac/code/log_ll.py", line 80, in info
    self.logger.info(msg, *args)
Message: '111'
Arguments: (111,)
--- Logging error ---
Traceback (most recent call last):
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 1100, in emit
    msg = self.format(record)
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 943, in format
    return fmt.format(record)
  File "/Users/mac/code/log_ll.py", line 21, in format
    log_message = super().format(record)
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 678, in format
    record.message = record.getMessage()
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 368, in getMessage
    msg = msg % self.args
TypeError: not all arguments converted during string formatting
Call stack:
  File "/Users/mac/code/log.py", line 50, in <module>
    pr.info("111", b)
  File "/Users/mac/code/log_ll.py", line 80, in info
    self.logger.info(msg, *args)
Message: '111'
Arguments: ((1, 2, 3),)
--- Logging error ---
Traceback (most recent call last):
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 1100, in emit
    msg = self.format(record)
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 943, in format
    return fmt.format(record)
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 678, in format
    record.message = record.getMessage()
  File "/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/logging/__init__.py", line 368, in getMessage
    msg = msg % self.args
TypeError: not all arguments converted during string formatting
Call stack:
  File "/Users/mac/code/log.py", line 50, in <module>
    pr.info("111", b)
  File "/Users/mac/code/log_ll.py", line 80, in info
    self.logger.info(msg, *args)
Message: '111'
Arguments: ((1, 2, 3),)
[22:15:48     INFO]: 111
[22:15:48     INFO]: 111
[22:15:48     INFO]: (1, 2, 3)
[22:15:48     INFO]: {'a': 1}
[22:15:48     INFO]: hello, 2
[22:15:48  WARNING]: hello 3
[22:15:48    ERROR]: hello error
[22:15:48    ERROR]: call stack:
  File "/Users/mac/code/log.py", line 58, in <module>
    pr.error("hello %s" % "error")

[22:15:48 CRITICAL]: hello, critical
[22:15:48 CRITICAL]: call stack:
  File "/Users/mac/code/log.py", line 59, in <module>
    pr.critical("hello, %s", "critical")
```
