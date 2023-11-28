---
layout: post
title: "chatgpt (resources)"
author: "twistfatezz"
date: 2023-05-20 11:20
categories: "2023"
tags:
  - chatgpt
---

### # chatgpt for terminal: aichat
The Dockerfile:
```text
# ref: https://github.com/sigoden/aichat
FROM rust
RUN cargo install --force aichat && \
    apt update && \
    apt install -y vim && \
    rm -rf /var/lib/apt/lists/*

# todo: copy local config.yaml to replace default /root/.config.yaml
# given a cmd or entrypoint to make this script work as cmd
```
need paied chatgpt account for api key tokens.
```text
docker build -t aichat .
```
<hr>

### # poe
https://poe.com/, using google account to login.

<hr>

### # forfront.ai
http://forefront.ai, using google account to login.
