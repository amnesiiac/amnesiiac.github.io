---
layout: post
title: "count lines of code in project (shell)"
author: "twistfatezz"
date: 2023-04-11 22:04
categories: "2023"
tags:
  - shell
---
### # count the lines of all files under current dir
```shell
# subdir excluded in the result, with total count provided
$ cd ~/nvim 
$ wc -l *
```
```txt
52 Dockerfile
wc: config: read: Is a directory
wc: scripts: read: Is a directory
52 total
```

<hr>

### # count the lines of each file under current dir
```shell
# subdir included in the result, but no total count provided
$ cd ~/nvim
$ find . -exec wc -l {} \; | sort -n
```
```txt
wc: .: read: Is a directory
wc: ./config: read: Is a directory
wc: ./config/.config: read: Is a directory
wc: ./config/.config/nvim: read: Is a directory
wc: ./config/.config/nvim/test: read: Is a directory
wc: ./config/.config/nvim/plugin: read: Is a directory
wc: ./config/.config/nvim/lua: read: Is a directory
wc: ./config/.config/nvim/lua/lsp: read: Is a directory
wc: ./config/.config/nvim/lua/lsp/config: read: Is a directory
wc: ./config/.config/nvim/lua/macro: read: Is a directory
wc: ./config/.config/nvim/lua/plugincfg: read: Is a directory
wc: ./scripts: read: Is a directory
       0 ./config/.DS_Store
       0 ./config/.config/.DS_Store
       1 ./config/.config/nvim/lua/macro/startup.lua
       2 ./.DS_Store
       4 ./scripts/entrypoint.sh
       5 ./config/.config/nvim/lua/macro/indentation.lua
       5 ./config/.config/nvim/lua/plugincfg/nvim-autopairs.lua
      10 ./config/.config/nvim/lua/plugincfg/indent-blankline.lua
      18 ./scripts/start.sh
      21 ./config/.config/nvim/lua/lsp/config/python.lua
      24 ./config/.config/nvim/lua/lsp/config/clangd.lua
      31 ./config/.config/nvim/lua/lsp/setup.lua
      36 ./config/.config/nvim/lua/plugincfg/nvim-treesitter.lua
      43 ./config/.config/nvim/lua/plugincfg/lualine.lua
      47 ./config/.config/nvim/lua/lsp/config/lua.lua
      49 ./config/.config/nvim/lua/plugincfg/nvim-tree.lua
      51 ./config/.config/nvim/init.lua
      52 ./Dockerfile
      67 ./config/.config/nvim/lua/plugincfg/which-key.lua
      72 ./config/.config/nvim/test/issues.md
      73 ./config/.config/nvim/lua/plugincfg/nvim-cmp.lua
      80 ./config/.config/nvim/lua/basic.lua
      83 ./config/.config/nvim/lua/plugincfg/fzf-lua.lua
      91 ./config/.config/nvim/lua/plugins.lua
     144 ./config/.config/nvim/plugin/packer_compiled.lua
     170 ./config/.config/nvim/lua/keybindings.lua
     524 ./config/.config/nvim/test/test.cpp
    1567 ./config/.config/nvim/test/test.c
    1611 ./config/.config/nvim/test/test.py
```

<hr>

### # count the lines of each file under current dir
# subdir included in the result, with total counted provided
```shell
$ cd ~/nvim
$ find . -exec wc -l {} + | sort -n
```
```txt
wc: .: read: Is a directory
wc: ./config: read: Is a directory
wc: ./config/.config: read: Is a directory
wc: ./config/.config/nvim: read: Is a directory
wc: ./config/.config/nvim/test: read: Is a directory
wc: ./config/.config/nvim/plugin: read: Is a directory
wc: ./config/.config/nvim/lua: read: Is a directory
wc: ./config/.config/nvim/lua/lsp: read: Is a directory
wc: ./config/.config/nvim/lua/lsp/config: read: Is a directory
wc: ./config/.config/nvim/lua/macro: read: Is a directory
wc: ./config/.config/nvim/lua/plugincfg: read: Is a directory
wc: ./scripts: read: Is a directory
       0 ./config/.DS_Store
       0 ./config/.config/.DS_Store
       1 ./config/.config/nvim/lua/macro/startup.lua
       2 ./.DS_Store
       4 ./scripts/entrypoint.sh
       5 ./config/.config/nvim/lua/macro/indentation.lua
       5 ./config/.config/nvim/lua/plugincfg/nvim-autopairs.lua
      10 ./config/.config/nvim/lua/plugincfg/indent-blankline.lua
      18 ./scripts/start.sh
      21 ./config/.config/nvim/lua/lsp/config/python.lua
      24 ./config/.config/nvim/lua/lsp/config/clangd.lua
      31 ./config/.config/nvim/lua/lsp/setup.lua
      36 ./config/.config/nvim/lua/plugincfg/nvim-treesitter.lua
      43 ./config/.config/nvim/lua/plugincfg/lualine.lua
      47 ./config/.config/nvim/lua/lsp/config/lua.lua
      49 ./config/.config/nvim/lua/plugincfg/nvim-tree.lua
      51 ./config/.config/nvim/init.lua
      52 ./Dockerfile
      67 ./config/.config/nvim/lua/plugincfg/which-key.lua
      72 ./config/.config/nvim/test/issues.md
      73 ./config/.config/nvim/lua/plugincfg/nvim-cmp.lua
      80 ./config/.config/nvim/lua/basic.lua
      83 ./config/.config/nvim/lua/plugincfg/fzf-lua.lua
      91 ./config/.config/nvim/lua/plugins.lua
     144 ./config/.config/nvim/plugin/packer_compiled.lua
     170 ./config/.config/nvim/lua/keybindings.lua
     524 ./config/.config/nvim/test/test.cpp
    1567 ./config/.config/nvim/test/test.c
    1611 ./config/.config/nvim/test/test.py
    4881 total
```
