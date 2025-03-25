---
layout: post
title: "rpmsg kdrv issue: scheduling while atomic (kdrv, rpmsg, sync)"
author: "melon"
date: 2025-03-04 20:24
categories: "2025"
tags:
  - driver
  - icc
  - kernel
---

this article mainly describe an issue: scheduling while atomic, caused by using mutex as sync promitive
in __dev_queue_xmit context of rpmsg kdrv implementation.

<hr>

### # issue logs & analysis

<hr>

### # brief introduction of rcu concept

<hr>

### # todo
