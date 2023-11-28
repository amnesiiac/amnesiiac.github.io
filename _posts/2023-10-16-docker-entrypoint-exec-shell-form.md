---
layout: post
title: "entrypoint script startup: shell/exec form (docker & dockerfile)"
author: "melon"
date: 2023-10-16 20:19
categories: "2023"
tags:
  - container
  - todo
---

### # introduction
1) two ways of using entrypoint script inside Dockerfile:  
   https://docs.docker.com/engine/reference/builder/#exec-form-entrypoint-example  
   https://docs.docker.com/engine/reference/builder/#shell-form-entrypoint-example

2) shell form (shell does the env var expansion not docker):  
   ENTRYPOINT [ "sh", "-c", "echo $HOME" ]  
   an examples to illustrate shell form usages: https://stackoverflow.com/a/37904830

3) exec form (does not invoke a command shell, no var substitution on $HOME):  
   ENTRYPOINT [ "echo", "$HOME" ]

ref: official document  
