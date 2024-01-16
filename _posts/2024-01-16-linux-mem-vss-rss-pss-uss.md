---
layout: post
title: "vss & rss & pss & uss (linux memory)"
author: "melon"
date: 2024-01-15 08:15
categories: "2024"
tags:
  - linux
---

### # intro
generally, the relationship between 4 memory statistics are as: vss >= rss >= pss >= uss.


### # vss (virtual set size)
definition: virtual statistics memory occupytion.

<hr>

### # rss (resident set size)
definition: rough physical memory occupytion.
for shared lib loaded into the mem, rss will just take all of them into account for each proc invoked the lib.
will add duplicate part mem when computing all procs mem usage.

<hr>

### # pss (proportional set size)
definition: approximately the real physical memory occupytion.
for shared lib loaded into the mem, pss will split them into different proc that invoked the lib.
which will be realy useful for computing total mem for all procs (no dup part considered).

<hr>

### # uss (unique set size)
definition: physical private memory taken up by proc sololy.

<hr>

### # memory model graph cross running proc with shared memory
given a live 5 processes memory model as follows:
```txt
    ┌─────────────────────────────────────────────────────────────────────────────────┐
    │                         memory occupied by shared libs                          │
    └─────────────────────────────────────────────────────────────────────────────────┘
    ┌──────────────────────────────────────┐   ┌──────────────────────────────────────┐
    │      process #1 allocated memory     │   │      process #2 allocated memory     │
    │ ┌─────────────┐ ┌──────────────────┐ │   │ ┌─────────────┐ ┌──────────────────┐ │
    │ │ free memory │ │ memory under use │ │   │ │ free memory │ │ memory under use │ │
    │ └─────────────┘ └──────────────────┘ │   │ └─────────────┘ └──────────────────┘ │
    └──────────────────────────────────────┘   └──────────────────────────────────────┘
    ┌──────────────────────────────────────┐   ┌──────────────────────────────────────┐
    │      process #3 allocated memory     │   │      process #4 allocated memory     │
    │ ┌─────────────┐ ┌──────────────────┐ │   │ ┌─────────────┐ ┌──────────────────┐ │
    │ │ free memory │ │ memory under use │ │   │ │ free memory │ │ memory under use │ │
    │ └─────────────┘ └──────────────────┘ │   │ └─────────────┘ └──────────────────┘ │
    └──────────────────────────────────────┘   └──────────────────────────────────────┘

         (no shared lib memory utilized)
    ┌──────────────────────────────────────┐
    │      process #5 allocated memory     │
    │ ┌─────────────┐ ┌──────────────────┐ │
    │ │ free memory │ │ memory under use │ │
    │ └─────────────┘ └──────────────────┘ │
    └──────────────────────────────────────┘
```
process 1-4 using common shared lib, while process 5 dont.  
let x denote proc 1-4, hence, the four memory can be computed as:  
vss_x = memory_allocated_x + common_shared_lib_memory
```txt
    ┌─────────────────────────────────────────────────────────────────────────────────┐
    │                         memory occupied by shared libs                          │
    └─────────────────────────────────────────────────────────────────────────────────┘
    ┌──────────────────────────────────────┐
    │      process #1 allocated memory     │
    │ ┌─────────────┐ ┌──────────────────┐ │
    │ │ free memory │ │ memory under use │ │
    │ └─────────────┘ └──────────────────┘ │
    └──────────────────────────────────────┘
```
rss_x = memory_under_use_x + common_shared_lib_memory
```txt
    ┌─────────────────────────────────────────────────────────────────────────────────┐
    │                         memory occupied by shared libs                          │
    └─────────────────────────────────────────────────────────────────────────────────┘
    ┌--------------------------------------┐
    |      process #1 allocated memory     |
    | ┌-------------┐ ┌──────────────────┐ |
    | | free memory | │ memory under use │ |
    | └-------------┘ └──────────────────┘ |
    └--------------------------------------┘
```
pss_x = memory_under_use_x + common_shared_lib_memory / num_proc_using_shared_mem
```txt
    ┌─────────────────────------------------------------------------------------------┐
    │ portion used by #1 |    memory occupied by shared libs                          |
    └─────────────────────------------------------------------------------------------┘
    ┌--------------------------------------┐
    |      process #1 allocated memory     |
    | ┌-------------┐ ┌──────────────────┐ |
    | | free memory | │ memory under use │ |
    | └-------------┘ └──────────────────┘ |
    └--------------------------------------┘
```
uss_x = memory_under_use_x
```txt
    ┌---------------------------------------------------------------------------------┐
    |                         memory occupied by shared libs                          |
    └---------------------------------------------------------------------------------┘
    ┌--------------------------------------┐
    |      process #1 allocated memory     |
    | ┌-------------┐ ┌──────────────────┐ |
    | | free memory | │ memory under use │ |
    | └-------------┘ └──────────────────┘ |
    └--------------------------------------┘
```
