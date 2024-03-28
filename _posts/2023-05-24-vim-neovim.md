---
layout: post
title: "dockerized nvim bootstrap script (neovim)"
author: "melon"
date: 2023-05-24 23:11
categories: "2023"
tags:
  - vim
  - ongoing
---

### # dockerfile for my nvim
use `docker build -t ${image:tag} .` to build this as base image.
```dockerfile
# FROM ubuntu:latest
FROM ubuntu:22.04
MAINTAINER melon <664634473@qq.com>

COPY scripts/ /scripts/
COPY config/ /root/         # todo: enable dynamic mount config inside

# general
ENV DEBIAN_FRONTEND=noninteractive
RUN apt update && \
    apt install -y git && \
    apt install -y make && \
    apt install -y cmake && \
    apt install -y automake && \
    apt install -y autoconf && \
    apt install -y pkg-config && \
    apt install -y gcc && \
    apt install -y g++ && \
    apt install -y fzf && \
    apt install -y ninja-build && \
    apt install -y gettext && \
    apt install -y libtool-bin && \
    apt install -y unzip && \
    apt install -y curl && \
    apt install -y ripgrep && \
    rm -rf /var/lib/apt/lists/*

# nvim=0.9
RUN git clone https://github.com/neovim/neovim && \
    cd neovim && make CMAKE_BUILD_TYPE=RelWithDebInfo && \
    make install

# ctags
RUN git clone https://github.com/universal-ctags/ctags.git && \ 
    cd ctags && \
    ./autogen.sh && \
    ./configure --prefix=/usr/local && \
    make && \
    make install

# packer
RUN git clone --depth 1 https://github.com/wbthomason/packer.nvim \
 ~/.local/share/nvim/site/pack/packer/start/packer.nvim

# nvim autocmd
# run packersync in cmd line (optional: for dockerfile automation)
RUN nvim --headless -c 'autocmd User PackerComplete quitall' -c 'PackerSync'

# reload highlight lualine after all packer load finished (optional)
# nvim -c "autocmd User PackerComplete quitall" -c "hi lualine_a_normal cterm=bold,reverse" \
# -c "hi lualine_a_inactive cterm=bold,reverse" -c "hi lualine_a_visual cterm=bold,reverse" \
# -c "hi lualine_a_insert cterm=bold,reverse" -c "hi lualine_a_command cterm=bold,reverse" \
# -c "hi lualine_a_buffers_inactive cterm=bold"

# entrypoint
RUN chmod +x /scripts/entrypoint.sh
ENTRYPOINT ["/scripts/entrypoint.sh"]
```

<hr>

### # /scripts/entrypoint.sh
```shell
#!/bin/sh
set -e

if [ "$1" = "nvim" ]; then
    shift
    nvim "$@"
fi

# exec "$@"
```

<hr>

### # /scripts/start.sh
save or create symbolic link under $PATH using the below script, then chmod +x.  
todo: enable running nvim container in background, and mount volatile file/dir under /mount to overwrite and edit, which will give a faster startup.
```shell
#!/bin/bash

# set base image
nvim_img="nvim:t"

# if no argument is passed, mount current dir
if [[ $# -eq 0 ]]; then
    dir="${PWD}"
    docker run -it --rm -v ${dir}:/root/mount/ ${nvim_img} nvim .
    exit 0
else
    cmd="nvim $@"
    option_str=""
    for arg in "$@"; do
        if [[ $arg == "--help" ]]; then  # option
            docker run -it --rm ${nvim_img} nvim --help
            exit 0
        elif [[ $arg == -* ]]; then  # option
            option_str+=" $arg"
            shift
        elif [[ $arg == --* ]]; then  # option
            option_str+=" $arg"
            shift
        elif [[ $arg == . ]]; then  # nvim .
            dir=${PWD}
            docker run -it --rm -v ${dir}:/root/mount/ ${nvim_img} nvim .
            exit 0
        else  # nvim -d -r file1 file2, nvim -r file1, nvim -r dir1, nvim -d dir1 dir2
            stuff="$@"
            mount_str=""
            file_str=""
            for item in "$@"; do
                if [ -f "$item" ]; then
                    absolute_path="$(cd "$(dirname "$item")"; pwd -P)/$(basename "$item")"
                    filename=$(basename "$item")
                    file_str+=" $filename"
                    mount_str+=" -v"
                    mount_str+=" $absolute_path:/root/mount/$filename"
                elif [ -d "$item" ]; then
                    if [[ $item == \.* ]]; then  # ./xxx/yyy
                        item="${PWD}/${item#./}"
                    elif [[ $item == \~* ]]; then  # ~/xxx/yyy
                        :
                    elif [[ $item == \/* ]]; then  # /xxx/yyy
                        :
                    fi
                    docker run -it --rm -v ${item}:/root/mount ${nvim_img} nvim .
                    exit 0
                else
                    echo "invalid input! format error or file/dir not existed!" && exit 1
                fi
            done
            docker run -it --rm${mount_str} ${nvim_img} nvim${option_str}${file_str}
            exit 0
        fi
    done
fi
```
