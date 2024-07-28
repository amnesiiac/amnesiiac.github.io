---
layout: post
title: "setup llm env (autodl, gpu)"
author: "melon"
date: 2024-04-09 21:01
categories: "2024"
tags:
  - llm
  - todo
---

script for autodl instance config for llm:
```text
#!/bin/sh

# for github.com, githubusercontent.com, githubassets.com, huggingface.co
source /etc/network_turbo

# llm project
cd /root
git clone https://github.com/karpathy/llm.c.git
mv llm.c llm
cd /root/llm && pip3 install -r requirements.txt

# modprobe
apt install kmod

# cuda
cd /root/autodl-tmp
wget https://developer.download.nvidia.com/compute/cuda/12.4.0/local_installers/cuda_12.4.0_550.54.14_linux.run
chmod +x ./cuda_12.4.0_550.54.14_linux.run
./cuda_12.4.0_550.54.14_linux.run  # install cuda interactively
```
