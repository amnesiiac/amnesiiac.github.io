---
layout: post
title: "commandline copy and paste (linux)"
author: "melon"
date: 2023-07-29 07:09
categories: "2023"
tags:
  - linux
---

### # pbcopy & pbpaste (macos)
todo


### # xclip & xsel (centos)
install the xclip & xsel on centos 7:
```text
sudo yum install xclip xsel   # need root for this
```
setup alias for same handle for copy/paste:
```text
$ which pbcopy
alias pbcopy='xclip -selection clipboard'
              /usr/local/bin/xclip

$ which pbpaste
alias pbpaste='xclip -selection clipboard -o'
               /usr/local/bin/xclip

$ source ~/.bashrc    # enable the command
```
