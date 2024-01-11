---
layout: post
title: "zombie process (linux)"
author: "melon"
date: 2024-01-07 13:41
categories: "2024"
tags:
  - linux
---

### # intro
zombie proc terminated before its parent proc, but prarent proc failed to release child proc
(wait: derive info about child proc, release relevant resources). the only way to wipe out
zombie proc is to destory its parent proc.

after any subproc (except for init) exit(), will not vanish at once, but left with a data struct
called zombie, waiting for parent proc to deal with, which is a common behavior at each subproc's ending.

if a child proc exits without proper disposal by its parent proc, then ps could show the state
of subproc as Z. if the parent proc is able to disposal its subproc in time,
we may not see the zombie stage of the subproc.

if the parent proc exit before subproc exit, then init will take over the ownership of subproc,
which will act as parent proc responsible for handling the release work of zombie subproc.

