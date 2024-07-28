---
layout: post
title: "chatgpt (llm, chatgpt, tool)"
author: "melon"
date: 2023-05-20 11:20
categories: "2023"
tags:
  - llm
---

### # aichat: chatgpt in terminal
the dockerfile:
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
need paid chatgpt account for api key tokens.
```text
docker build -t aichat .
```

<hr>

### # poe
https://poe.com/, using google account to login.

<hr>

### # forefront.ai
http://forefront.ai, using google account to login.
