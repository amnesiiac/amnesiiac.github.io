---
layout: post
title: "reverse ssh tunnel for remote debug (ssh, reverse tunnel)"
author: "melon"
date: 1111-03-05 21:03
categories: "2025"
tags:
  - python
---

the following diff introduces a python function, to enable ssh login on remote machine in a hack way,
in order to facilitate remote debug on all machine running the hostfw instance.

```blurtext
diff --git a/hostfw/start-ls-host-by-batch.py b/hostfw/start-ls-host-by-batch.py
--- a/hostfw/start-ls-host-by-batch.py
+++ b/hostfw/start-ls-host-by-batch.py
@@ -103,14 +103,6 @@ def protect_args(obj, arg, name):
     obj[name] = getattr(arg, name)
 
 
-# use for debug online in buildme
-def remote_debug():
-    # sh.run("echo 'ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAtBI6dVwnl3nOsazV/OHxWTuc9SAF24q2xzl6cTwf51ErgXfzTlx
WEgAWyrOcnpvIrOPpACbiiTPupB6nOWGn82mRWZxlc4JZpYpa8sJqIPuSFw5kyk1O90mDkwby9YcgyJgDwsU59m6hhp6S51FJmZvL7iJDCYi
LptO6XBlKarNSIDb3zzhh8SVm3bNp5O1MfxDlPkxOb203R8oVl4k+njF7reprQ1fwdTL53h4MozDxuRDH9wc6t+Fwx2fK8ZhlhwMxbLpoeXm
lIi/fTJ6vPBOr5MItYgdDLtNH6reiRSnjEQivWQjMBzY4vMgeAPDrNsS5P0D/pozCsYhuOCFbnw== debug' >> $HOME/.ssh/authorized_keys")
-    # sh.run("sshpass -p xxx ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@10.182.101.102
sh -c 'echo ok > /tmp/remote_clicktest_5200'")
-    # pt.thread_run(sh.run, "sshpass -p xxx ssh -N -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
-R 5200:localhost:22 root@10.182.101.102")
-    pass
-
-
 if __name__ == "__main__":
     # 1. don't insert anything into this arg parsing block! -h should depend on nothing and return very fast!
     # parse_batch_case_file("/repo1/team/robot/ATS/MOSWA/COMMON/COMMON_SERVICES/IPPROXY/BATCH/FI_CFNTB_IPPROXY.json", "temp")
@@ -337,9 +329,6 @@ if __name__ == "__main__":
     #                 ltb_path,
     #                 errno=Error.err_testfw_cfg_error)
 
-    # 7. start remote debug port
-    remote_debug()
-
     args.build = dhbl.get_final_build(cfg.DAILY_BUILDS_ARTIFACTORY_URL, args.build)
     pr.info("Build[FINAL]: build=%s", args.build)
     build, _ = dhbl.get_build_info(args.build)
```

the above diff set up a reverse ssh tunnel, from the machine running hostfw instance back to local server
10.182.101.102, allowing local host at 10.182.101.102 to access the remote machine's ssh service (port 22)
through the tunnel on port 5200.
