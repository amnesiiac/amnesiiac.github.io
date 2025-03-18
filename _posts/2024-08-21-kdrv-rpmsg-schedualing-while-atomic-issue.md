---
layout: post
title: "rpmsg kdrv issue: scheduling while atomic (kdrv, rpmsg, sync)"
author: "melon"
date: 2024-08-21 21:00
categories: "2024"
tags:
  - driver
  - virtualization
  - icc
  - todo
---

this article mainly describe an issue: scheduling while atomic, caused by using mutex as sync promitive
in __dev_queue_xmit context of rpmsg kdrv implementation.

the detailed rpmsg kdrv implementation can be found at blog post: inter-core communication on amp arch II:
the target part.

<hr>

### # issue logs & analysis

<hr>

### # brief introduction of rcu concept

<hr>

### # todo
