---
layout: post
title: "generate random string (shell)"
author: "twistfatezz"
date: 2023-04-11 22:58
categories: "2023"
tags:
  - shell
---

### # method 1: md5 checksums hash
bash has an env variable "RANDOM" which provide a random number
```shell
$ echo $RANDOM
```
```txt
6230
```
thus we could pipe the "RANDOM" to md5 hash to get a random string
```shell
$ echo $RANDOM | md5sum | head -c 5; echo;
```
```txt
f035f
```

<hr>

### # method 2: kernel UUID generator
```shell
$ cat /proc/sys/kernel/random/uuid    # generate a unique hexadecimal value
```
thus we could generate a random string using sed command
```shell
$ cat /proc/sys/kernel/random/uuid | sed 's/[-]//g' | head -n 5; echo;
```

<hr>

### # method 3: pseudo devices
Files located under '/dev' are known as pseudo devices, which are bridges between kernel & software. 
One of them named 'urandom' file which provide an interface to kernel's random number generator.

Hence, we could pipe the random number to 'tr' to gnerate random string:
```shell
$ cat /dev/urandom | tr -dc '[:alpha:]' | fold -w ${1:-20} | head -n 1
```
```txt
qzaUdkCKjwOqAcPXMkFS
```
can get multiple of them at the same time:
```shell
$ cat /dev/urandom | tr -dc '[:alpha:]' | fold -w ${1:-20} | head -n 3
```
```txt
iGGUYMGUrbeDdOaieSWa
JiJRzoVPEIoXkBTHYuYV
bQunaVHcaYkUKqcFgaqi
JUChKngzxYyIUDAMIAQH
```
in macos, the above command does not work, should set LOCALE before usage:
```shell
$ LC_CTYPE=C tr -dc '[:alpha:]' < /dev/urandom | fold -w ${1:-20} | head -n 4
$ LC_ALL=C tr -dc '[:alpha:]' < /dev/urandom | fold -w ${1:-20} | head -n 4
```
```txt
PWvAKChjbzvlJoNZOCdO
RKVHCiHesrIHfDdEWHdj
pWjNLQWdwmscEsQUETid
htLCusUFRXspRlZVGDOO
```

<hr>

### # method 4: base64
```shell
$ echo $RANDOM | base64 | head -c 20; echo;
```
```txt
MjgwMzEK
```
<hr>

### # method 5: openssl pseudo random bytes
OpenSSL rand command allows you to generate random bytes based on the type specified.
```shell
$ openssl rand -hex 10
```
```txt
4d2f7c8b60cf12364218
```
```shell
$ openssl rand -base64 10
```
```txt
rnZJz97luwKahA==
```
