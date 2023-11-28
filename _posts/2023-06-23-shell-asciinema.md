---
layout: post
title: "asciinema: record terminal actions (shell)"
author: "melon"
date: 2023-06-23 22:26
categories: "2023"
tags:
  - shell
---

### # asciinema usages on macos
#### \>>> installation:
```shell
$ brew install asciinema
```

#### \>>> record
record & save to local:
```shell
$ asciinema rec ${filename}.cast
```
record & upload to remote:
```shell
$ asciinema rec -t "${my_record_name}"
```

#### \>>> play
play local record:
```shell
asciinema play ${record.cast}
```
play remote cast:
```shell
$ asciinema play https://asciinema.org/a/difqlgx86ym6emrmd8u62yqu8
```

<hr>

### # asciinema --help
```text
$ asciinema --help
usage: asciinema [-h] [--version] {rec,play,cat,upload,auth} ...

Record and share your terminal sessions, the right way.

positional arguments:
  {rec,play,cat,upload,auth}
    rec                 Record terminal session
    play                Replay terminal session
    cat                 Print full output of terminal session
    upload              Upload locally saved terminal session to asciinema.org
    auth                Manage recordings on asciinema.org account

optional arguments:
  -h, --help            show this help message and exit
  --version             show program's version number and exit

example usage:
  Record terminal and upload it to asciinema.org:
    asciinema rec
  Record terminal to local file:
    asciinema rec demo.cast
  Record terminal and upload it to asciinema.org, specifying title:
    asciinema rec -t "My git tutorial"
  Record terminal to local file, limiting idle time to max 2.5 sec:
    asciinema rec -i 2.5 demo.cast
  Replay terminal recording from local file:
    asciinema play demo.cast
  Replay terminal recording hosted on asciinema.org:
    asciinema play https://asciinema.org/a/difqlgx86ym6emrmd8u62yqu8
  Print full output of recorded session:
    asciinema cat demo.cast

For help on a specific command run:
  asciinema <command> -h
```

<hr>

### # convert asciinema cast file to gif
github repo: https://github.com/dstein64/gifcast  
online converter tool: https://dstein64.github.io/gifcast/

<hr>

### # reference
https://asciinema.org  
https://asciinema.org/docs

