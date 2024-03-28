---
layout: post
title: "ebpf: extended berkelay packet filter (linux)"
author: "melon"
date: 2024-01-01 20:16
categories: "2024"
tags:
  - linux
  - kernel
  - ebpf
  - todo
---

### # introduction
```txt

         ┌───────────────┐
         │  user mode    │
         │ applications  │
         ├───────────────┤
         │ system calls  │
         ├───────────────┼──────────────────┐
         │               │  kernel mode     │
         │               │ bpf applications │
         │               ├──────────────────┤
         │               │ bpf helper calls │
         │               └──────────────────┤
         │              kernel              │
         ├──────────────────────────────────┤
         │              driver              │
         ├──────────────────────────────────┤
         │             hardware             │
         └──────────────────────────────────┘
```
ref: https://eunomia.dev/zh/tutorials/0-introduce/
