---
layout: post
title: "decorator (python)"
author: "melon"
date: 2023-10-21 20:07
categories: "2023"
tags:
  - python
---

### # a toy code to illustrate the decorator usage
```text
def decorator(func):
    def wrapper(*args, **kwargs):
        print("executing decorator code before the func")  # code to be executed before the original function
        result = func(*args, **kwargs)                     # call the original function
        print("executing decorator code after the func")   # code to be executed after the original function
        return result                                      # return res of original function
    return wrapper

@decorator
def my_function():
    print("executing the original func")

my_function()                                              # invoke decorated func
```

using python 3.7.0 to run the toy code above, output:
```text
executing decorator code before the func
executing the original func
executing decorator code after the func
```
the decorator name can be customized according to real needs.

<hr>

### # code in action
the robot repo is a test framework basically cover the test case of traffic validation & netconf
& pcta (opensource stream sender).  
the usages of decorator inside test framework:
```text
...
def decorator(func):
    def wrapper(*args, **kwargs):
        NEW_TEST = logger.get_test_name()
        log_dir = logger.get_log_dir()
        kw_name = func.__name__

        dKwCount[kw_name] = dKwCount[kw_name] + 1
        if not dKwTest[kw_name] == NEW_TEST:
            dKwCount[kw_name] = 1
            dKwTest[kw_name] = NEW_TEST

        robotlogger.info('<a href=\"%s/%s.html#%s\">%s</a>' % (log_dir,
            NEW_TEST, kw_name + '.' + str(dKwCount[kw_name]), kw_name), html=True)

        try:
            logger.start_kw(NEW_TEST, kw_name + '.' + str(dKwCount[kw_name]))
            res = func(*args, **kwargs)
            logger.end_kw(NEW_TEST, kw_name + '.' + str(dKwCount[kw_name]))
            return res
        except Exception as inst:
            raise PctaContinueError("%s " % inst)
    return wrapper

...

@decorator
def disconnect_pcta():
    # disconnect pcta by close pcta session
    global PCTA_STATUS
    if PCTA_STATUS.lower() == 'false':   # if PCTA is not available, exit from this function
        logger.info("PCTA is not available, exit from this function!")
        return 'PASS'

    global PCTA_SESSION
    keyword_name = "disconnect_pcta"
    if not PCTA_SESSION: return
    try:
        PCTA_SESSION.close_pcta()
        del PCTA_SESSION
    except Exception as inst:
        raise AssertionError("%s:%s-> fail to close pcta session, exception: %s" % (__name__, keyword_name, str(inst)))
    else:
        logger.info("disconnect pcta connection success ")
    logger.set_close_flag()
```

from the above code, we could see that the decorator is used to separate logger formatter logic code
out of the real test case logic code.  
the decorated log output is used by some other log.html visualizer engine to parse and present.

ref: pcta_command wrapper script file in robot repo -> robot/LIBS/COM_PCTA/pcta_command.py
