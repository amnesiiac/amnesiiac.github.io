---
layout: post
title: "timeout usages (shell)"
author: "melon"
date: 2023-12-25 22:12
categories: "2023"
tags:
  - shell
---

### # try adding timeout for whileloop / forloop
```text
$ touch test

$ tail -f test | grep -e "package" &

$ jobs
[1]+  Running                 tail -f test | grep --color=auto -e "package" &

$ timeout 5 bash -c -- 'while true; do echo "installing packages" >> test 2>&1; sleep 1;  done'
installing packages
installing packages
installing packages
installing packages
installing packages
```

explanation for the above timeout usage:  
```text
1 bash -c mean: then commands are read from the first non-option argument command_string.
2 -- assures that the following arguments will be treated as non-option.  
3 '' helps with passing " without unnecessary escaping.
```

<hr>

### # an equivalent form (precision loss)
```text
$ for i in {1..10}; do echo "installing packages" >> test 2>&1; sleep 1; done
installing packages
installing packages
installing packages
installing packages
installing packages
installing packages
installing packages
installing packages
installing packages
installing packages
```

<hr>

### # failures for using timeout for shell cmds:
failed attempts to add timeout for whileloop:
```text
$ timeout 5 while TRUE; do printf "."; done
-bash: syntax error near unexpected token `do'

$ timeout 5 "while TRUE; do printf "."; done"
timeout: failed to run command ‘while TRUE; do printf .; done’: No such file or directory

$ timeout 5 "while TRUE; do printf \".\"; done"
timeout: failed to run command ‘while TRUE; do printf "."; done’: No such file or directory

$ timeout 5 $(while TRUE; do printf "."; done)
-bash: TRUE: command not found
Try 'timeout --help' for more information.
```
