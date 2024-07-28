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

note: need to config docker daemon.json proxy or will meet timeout in connection.  
cmd to buildup the image:

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

<hr>

### # todo
migration the blog to alpine linux.
