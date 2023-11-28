---
layout: post
title: "resume latex template dockerfile env (latex)"
author: "melon"
date: 2023-09-14 21:57
categories: "2023"
tags:
  - latex
  - container
---

### # dockerfile to buildup texlive-full env
```text
# ref: https://github.com/sb2nov/resume
FROM ubuntu:xenial
ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update -q && apt-get install -qy \
    curl jq \
    texlive-full \
    python-pygments gnuplot \
    make git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /data
VOLUME ["/data"]
```
build up docker image using:
```text
$ docker build -t latex:latest . 
```
note: the texlive-full is extremely large, proxy for downloading docker image is recommanded.

<hr>

### # methods to buildup resume
use xelatex extension to build up chinese/english resume:
```text
$ cd /Users/mac/resume/
$ ls 
Dockerfile    LICENSE    README.md    
en.aux        en.log     en.out     en.pdf    en.tex
zh.aux        zh.log     zh.out     zh.pdf    zh.tex

# compile and generate pdf using zh.tex
$ docker run --rm -i -v "$PWD":/data latex xelatex ${zh.tex}

# compile and generate pdf using en.tex
$ docker run --rm -i -v "$PWD":/data latex pdflatex ${en.tex}
```
