---
layout: post
title: "vim - spacevim config"
subtitle: 'spacevim config for mac'
author: "twistfatezz"
# header-style: text
header-img: "img/in-post/vim_img/spacevimbg.jpg" 
header-mask: 0.6 
comments: false 
mathjax: false 
date: 2021-05-25 20:59 
lang: ch 
catalog: true 
categories: vim 
tags:
  - Time 2021
---

## SpaceVim Install & Config
### Download neovim for better use of spacevim
**[1.1]** Download nvim-macos.tar.gz from [url](https://github.com/neovim/neovim/releases/tag/v0.4.4) <br>
**[1.2]** Extract nvim using command: `tar xzvf nvim-macos.tar` <br>
**[1.3]** Run command: `./nvim-osx64/bin/nvim` <br>


### Download spacevim in terminal using command
`curl -sLf https://spacevim.org/cn/install.sh | bash`


### Set colortheme settings
**[3.1]** Open shuttle and goto the nvim config file to add ... to customize theme.


### Set the nvim response timeout for pop-up window
**[4.1]** Open shuttle and goto the nvim config file to add ... to customize timeout settings.


### Add configs for c/c++
**[5.1-lsp]** `Space+f+v+d` to open spacevim config file & add [[layers]] for clangd. <br>
Mention that cland must be installed to enbale `lsp` plug-in to work fine. <br>
First, go to [url](https://clangd.llvm.org/installation.html) to download clangd using: `brew install llvm`. <br>
Second, check in terminal: `which clangd` or `whereis clangd` or `clangd --verison`. <br>
Third, if nothing found, then the clangd should be in this dir: `/usr/local/opt/llvm/bin/clangd`. <br>
**[5.2-ycm]** Although `ycm` in [[layers]]-autocompelete can also dealwith c/c++, but i think `lsp` is better, so in spacevim config file, it's been commented out. <br>
**[5.3-lang#c]** `Space+f+v+d` to open spacevim config file & add [[layers]] for lang#c.


### Add configs for markdown
**[1]** `Space+f+v+d` to open spacevim config file & add [[layers]] for lang#markdown. <br>
**[2]** For markdown preview, Just press `Space+l+p` to preview it in localhost. More: input `:MarkdownPreview` also works.<br>


### Config SpaceVim for Ctags&Tagbar
**[No.1 ctags-excuberant install (for tagbar-plugins)]** <br>
Original Ctags cannot be used in SpaceVim in macos original terminal, what tagbar need to use is exuberant-ctags. <br>
**[1]** Run:`brew install ctags-exuberant` in terminal. <br>
**[2]** Check ctags-exuberant install dir: `/usr/local/Cellar/ctags/5.8_2/bin/ctags`. <br>
**[3]** Add lines in nvimrc: `let g:Tlist_Ctags_Cmd='/usr/local/Cellar/ctags/5.8_2/bin/ctags'` <br>
Then, restart spacevim, you can use F2 to see ctags for functions. If confused about the usage of ctags, try command `:help tagbar` to see more info.

**[No.2 tagbar customization]**<br>
The tagbar plugin is forked from [url](https://github.com/preservim/tagbar).
Original tagbar when open large file, the cursor movement on generated tags is quite laggy☹️.. So i just move some lines to fix it: <br>
**[1]** In nvim config file, Just swap`let g:tagbar_iconchars = ['+', '-']`with`let g:tagbar_iconchars = ['', '']`, which shutdown the tagbar icon can liberate some iterations to speedup the tagbar. <br>
**[2]** Once [1] is done, reopen tagbar and errors may occur like:
<center><img src="/img/in-post/vim_img/spacevim_1.jpg" width="80%"></center>
**[3]** To fix the error above: Goto `/Users/mac/.spacevim/bundle/tagbar/syntax/tagbar.vim` and commented the three line in error-pic out. Then u'll see no errors😅. <br>
**[4]** If u want to change the icon of class public、protected、private members, just go `/Users/mac/.spacevim/bundle/tagbar/autoload/tagbar/prototypes/basetag.vim` to change the `let s:visibility_symbols`dict, and goback to `/Users/mac/.spacevim/bundle/tagbar/syntax/tagbar.vim`and change the `TagbarVisibility...[access-sepcifier]` with your pre-defined one. All done! <br>
**[5]** If u want to change the color of class public、protected、private members, just go `/Users/mac/.spacevim/bundle/tagbar/syntax/tagbar.vim` and edit related lines. 

**[no.3 tagbar updatetime config]** <br>
If cursor over a certain tag, the information(function type) will echo out. According to the tagbar script below at `/Users/mac/.spacevim/bundle/tagbar/autoload/tagbar.vim`, the autocmd cursorhold timeout is defined by updatetime:
```c++
if !g:tagbar_silent
    autocmd CursorHold __Tagbar__.* call s:ShowPrototype(1)
endif
```
So, I just insert `set updatetime=20` in nvim config files to speed up the echo.


### Nerd-fonts implement
If want nerd-font for your nvim-spacevim interface, should try nerd-fonts, it add many aequilatus fonts.
~~I Only like menlo in macos though.~~ Now i switch to Meslo LG M DZ Bold Italic Nerd Font Complelte Mono 14. <br>
**[1]** `git clone https://github.com/ryanoasis/nerd-fonts.git --depth 1` <br>
**[2]** `cd nerd-fonts` <br>
**[3]** `./install.sh` <br>
**[4]** `cd ..` <br>
**[5]** `rm -rf nerd-fonts` <br>


### Syntax Attribute Inspector
For a deep vim color engineer, one may endure the confusion about in which scope(group) is the var/functions lies. So Just implement this plug-in to auto check. <br>
**[1]** Find the vim-plugin in [url](https://www.vim.org/scripts/script.php?script_id=383) or [github-url](https://github.com/vim-scripts/SyntaxAttr.vim). <br>
**[2]** Then open the spacevim rc file: input [[custom_plugins]]...<br>
**[3]** Reload nvim & spacevim to auto download & install SyntaxAttr.vim. <br>
**[4]** `:call SyntaxAttr()` to inpect the syntax scope(group) attributes(for color scheme customization). 


### Syntax Theme Customization
Color Theme related things can never be done completely. And one can never satisfied at the current theme settings. Here are some steps to customize the spacevim theme. <br>
**[1]** Find the correct syntax attribute with `:call SyntaxAttr()` command or just safari the [vim-color-pallete](/vim/2021/05/27/post-vim-spacevim-color-pallete/) to makesure the certain var/func for ajustment. <br>
**[2]** Customize the color background(bg) and color foreground(fg) using
`:hi[ghlight] [default] {group-name} {key}={arg}` in which the {group-name} and {key} can be found in awesome-pallete above. <br>
**[3]** If you are regretful about the ajustment you have done, just type `:hi[ghlight] clear` to reset all to the defaults.


### Leetcode on Vim Customization
Vim plugin from [github-repo](https://github.com/ianding1/leetcode.vim) can let one code for leetcode problems in local vim env. Here are some customizations done for better usage of it: <br>
**[1]** Find the plugin in [url](https://github.com/ianding1/leetcode.vim) <br>
**[2]** Then open the spacevim rc file: input [[custom_plugins]]...<br>
**[3]** Reload nvim & spacevim to auto download & install leetcode.vim. <br>
**[4]** Open Nvim config file to custom settings & keybindings. <br>
**[5]** Nvim the file `/Users/mac/.cache/vimfiles/repos/github.com/ianding1/leetcode.vim/autoload/leetcode.py` according to the lines begins with `"melon add ...` <br>
**[6]** Nvim the file `/Users/mac/.cache/vimfiles/repos/github.com/ianding1/leetcode.vim/autoload/leetcode.vim` according to the lines begins with `"melon add ...` <br>
All configuration are done.


## SpaceVim Usages
### Show Vim Keybindings
**[1]** `:help key-notation` to show the vim key bindings chart: \<CR\> enter...


### Toggle File Tree
**[1]** `Space+f+t` or `F3` to toggle file tree.


### Tagbar
**[1]** Goto spacvim-statusline config file & find: `SpaceVim#layers#core#statusline#_current_tag()`. If comment the 4lines out, no function tags will show on spacevim statusline.


### Toggle ctags-exuberant(tagbar)
**[1]** `F2` to toggle ctags-exuberant. <br>
**[2]** `Space+f+d` to toggle the ctags-exuberant(tagbar). <br>


### Word Wrap Toggles
**[1]** `Space+t+W` to toggle word warp settings.


### MarkDown Preview
**[1]** `Space+l+p` or `:MarkdownPreview` to preview md files.


### Inspect Syntax Attribute
**[1]** `:call SyntaxAttr()` to inpect the syntax scope(group) attributes(for color scheme customization). 


### Toggle highlight indentation lines
**[1]** `Space+t+h+i` to toggle the code indentation lines.


### Toggle highlight cursor current column/row
**[1]** `Space+t+h+c` to toggle current cursor column. <br>
**[2]** `Space+t+h+h` to toggle current cursor row.


### Toggle Syntax Highlight
**[1]** `Space+t+s` to toggle syntax highlight.


### Toggle UndoTree
**[1]** `F5` to toggle the undotree. 


### Toggle Neomake
**[1]** `Space+t+s` to toggle the neomake plugin which automatically check syntax errors after save file.


### core#banner
**[1]** `:help banner` to show the manul for plugin core#banner. <br>
**[2]** Goto `.spacevim/autoload/SpaceVim/layers/core//banner.vim` to edit the vim script & design the banner pattern u want.


### core#statusline
**[1]** Goto `.spacevim/autoload/SpaceVim/layers/core/statusline.vim` to edit the vim script & ajust the style u want.


### core#tabline
**[1]** Goto `.spacevim/autoload/SpaceVim/layers/core/tabline.vim` to edit the vim script & ajust the style u want.


### Leaderf fuzzy search
**[1]** `Leader+f+f` to open popup window for recent files. <br>
**[2]** `Leader+f+o` to open popup window for outlines/functions. <br>


### Defx related 
**[1]** `<` to narrow the filetree window size. <br>
**[2]** `>` to enlarge the filetree window size. <br>
**[3]** `N` to touch a file under the cursor. <br>
**[4]** `P` to paste a file under the cursor. Need xclip-copyfile in ur PATH. <br>
**[5]** `yy` to copy the file path to clipboard. <br>
**[6]** `yY` to copy the file itself to clipboard. <br>
**[7]** `.` to toggle the hidden files. <br>
**[8]** `ctrl+r` to refresh the file window 

### Autocomplete
Attention: <br>
For example: original path = `\Users\mac\document\books`, when u tap `|Users\mac\document`, the candidate items may like `books [F]`、`|Users\mac\document\books [FI]`, if u tab choose the first one, it may complete the path like `\Users\mac\document|Users\mac\document\books`, which has duplicated part of the implicit path.<br>
To avoid this, u should continue input `Uncle S`, then only one candidate item in drop-down menu, continue press Enter will get the right result. <br>
**[1]** `Space+t+a` to toggle the spacevim default autocomplete.

## Reference
> https://spacevim.org

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
