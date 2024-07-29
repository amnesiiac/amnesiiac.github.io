---
layout: post
title: "dockerfile for jekyll blog setup env (blog)"
author: "melon"
date: 2023-04-14 21:42
categories: "2023"
tags:
  - blog
---

for a freeze & stable env snapshot to build up the reliance for my jekyll blog,
a dockerfile is always the first choice: portable & small & extensible.

<hr>

### # dockerfile
1 take ubuntu:22.04 as the base layer of jekyll blog engine image (obsolete).

```text
FROM ubuntu:22.04
MAINTAINER melon <664634473@qq.com>

# remove Gemfile.lock to let the bundle automatically check the dependencies
COPY blog/Gemfile blog/Gemfile.lock /paper/

# ref: https://jekyllrb.com/docs/installation/ubuntu/
RUN apt update && \
	apt install -y ruby-full && \
	apt install -y build-essential && \
	apt install -y zlib1g-dev && \
	rm -rf /var/lib/apt/lists/*

RUN echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc && \
    echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc && \
    echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc && \
    . ~/.bashrc

RUN gem install jekyll bundler && \
	cd /paper/ && \
	bundle install
```

<p style="margin-bottom: 20px;"></p>

2 take alpine-3.20 as the base layer of jekyll blog engine image (active).

```text
FROM alpine:3.20
MAINTAINER melon <664634473@qq.com>

# proxy to accelerate external resource download at corp.
ENV https_proxy http://10.144.1.10:8080/
ENV http_proxy http://10.144.1.10:8080/

# remove Gemfile.lock when adjust this to new env
COPY Gemfile Gemfile.lock /paper/

# ref: https://jekyllrb.com/docs/installation/alpine/
RUN apk update              && \
    apk add -f ruby-full    && \
    apk add -f ruby-dev     && \
    apk add -f build-base   && \
    apk add -f zlib-dev     && \
    rm -rf /var/cache/apk/*

# add gem tool to $PATH
RUN echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc && \
    echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc && \
    echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc && \
    . ~/.bashrc

# gem install && bundle install (will generate Gemfile.lock)
RUN gem install jekyll bundler && \
    cd /paper/                 && \
    bundle install

# docker build -f ./dockerfile -t blog:latest .
```

note: for downloading timeout problem, need to add ENV proxy or config docker daemon.json proxy.

<p style="margin-bottom: 20px;"></p>

3 buildup the wholesome docker image by:

```text
$ docker build -f ./dockerfile -t blog:latest .
```

<hr>

### # jekyll blog startup

```text
$ docker run -it --rm -p 3000:4000 -w=/paper -v /Users/mac/paper/blog/:/paper \ 
-h melon blog bundle exec jekyll serve --host 0.0.0.0
```

which will start a blog instance with 3000 port on localhost exposed.
