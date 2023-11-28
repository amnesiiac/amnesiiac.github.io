---
layout: post
title: "run command made up from variables (shell)"
author: "melon"
date: 2023-08-24 22:15
categories: "2023"
tags:
  - shell
---

### # problem when startup qemu
```text
cmd to startup qemu inside bash script
```
the root cause for the above failure on qemu cmdline startup is that: unwisely relying on the splitting of unquoted variable to tokenize the space-separated string into individual arguments.  
an simple example of unreliable result derived by default space-separated string into arguments: 
```text
str="foo bar"
echo $str
```
```txt
foo bar
```
```text
printf '%s\n' $str  # default IFS is space
```
```txt
foo
bar
```
```text
IFS=$'\n\t'         # set IFS as newline+tab
printf '%s\n' $str
```
```txt
foo bar
```

<hr>

### # IFS (interal field separator)
The IFS indicates how the words are separated on the command line. By default the IFS in shell will be a space.  
It is used by the shell to determine how to do word splitting, i. e. how to recognize word boundaries.

shell comamnd to examine the IFS of env:
```text
echo "$IFS" | cat -et
```

an example to see the function of IFS:
```text
mystring="foo:bar baz rab"
for word in $mystring; do echo "$word"; done
```
default IFS=" ", will output:
```txt
foo:bar
baz
rab
```
set IFS as ":"
```text
IFS=":"
mystring="foo:bar baz rab"
for word in $mystring; do echo "$word"; done
```
the output is like:
```txt
foo
bar baz rab
```
if set IFS to empty string, the bash will not perform any splitting operation:
```text
IFS=""
mystring="foo:bar baz rab"
for word in $mystring; do echo "$word"; done
```
```txt
foo:bar baz rab
```

<hr>

### # solutions (run cmd with params stored inside variables)
(1) do not use eval in shell, which may introducing arbitrary code execution (major risk).  
(2) do not run the cmd directly as follows:
```text
abc='ls -l "/tmp/test/my dir"'
$abc
```
```txt
ls: cannot access '"/tmp/test/my': No such file or directory
ls: cannot access 'dir"': No such file or directory
```
(3) call the cmd inside a function (when no variables are expanded by shell)
```text
func_ls() {
    ls -l "/tmp/test/my dir"
}

func_ls
```
(4) call the cmd with parameters organized inside shell array:
```text
# define the arr, the 3rd params will be kept exactly it was even has spaces (IFS=space)
mycmd=(ls -l "/tmp/test/my dir")

# expand the arr and run the cmd
"${mycmd[@]}"
```

<hr>

### # reference
ref: https://unix.stackexchange.com/a/690292  
ref: https://stackoverflow.com/a/44055875  
ref: https://unix.stackexchange.com/a/444949


