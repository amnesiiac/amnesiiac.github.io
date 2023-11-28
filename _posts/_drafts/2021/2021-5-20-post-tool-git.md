---
layout: post
title: "Tool - git"
subtitle: 'git基本概念相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-20 16:50
lang: ch 
catalog: true 
categories: tool 
tags:
  - Time 2021
---

### Git
Git是世界上最先进的分布式版本控制系统。什么是版本控制？版本控制(Revision control)是一种在开发的过程中用于管理我们对文件、目录或工程等内容的修改历史，方便查看更改历史记录，备份以便恢复以前的版本的软件工程技术。Git能够非常高效方便的用于管理多人协同开发项目的技术。

### Git的四个操作区域
<center><img src="/img/in-post/tool_img/git_1.pdf" width="100%"></center>

### Git的五种编辑状态
<center><img src="/img/in-post/tool_img/git_2.pdf" width="100%"></center>

### Git配置
```python
# 显示 config 的配置 加--list
# 优先级：local > global > system
git config --list --local   # local 的范围是某个仓库
git config --list --global  # global 的范围是登录的用户
git config --list --system  # system 的范围是系统所有登录的用户
​
# 配置用户 name 和 email
git config --global user.name 'your_name '
git config --global user.email 'your_email@domain.com'
​
# 清除配置信息
git config --unset --global user.name
```

### Init仓库
```python
# 在当前目录初始化一个git仓库
git init
# 在当前路径下创建名为project_name的git仓库
git init project_name 
```

### Git Add
```python
git add <filename>  # 将特定文件添加到暂存区(index区)
# 见下面的表格整理
git add .  
git add -u
git add -A 
```
**For Git Version 1.x Changes can be staged by.**

| command | New Files | Modified Files | Deleted Files  
| :---: | :---: | :---: | :-:
| `git add -A` | √ | √ | √
| `git add .` | √ | √ | × 
| `git add -u` | × | √ | √

**For Git Version 2.x Changes can be staged by.**

| command | New Files | Modified Files | Deleted Files  
| :---: | :---: | :---: | :-:
| `git add -A` | √ | √ | √
| `git add .` | √ | √ | √
| `git add -u` | × | √ | √

### Git Status
```python
git status  # 参看workspace以及index区域(暂存区)的文件状态
```

### Git Commit
```python
git commit  # commit  
git commit -m '...'  # -m指定本次提交的注释信息 
```

### Git mv
```python
git mv old_name new_name  # 对于git仓库中的文件进行重命名
```

### Git rm
```python
#1 从git管理的文件中删除某个文件并把更改提交到index区 
git rm <filename> 
#2 从git管理的文件中删除某个文件夹并将更改提交到index区 
git rm -r <foldername> 

#3 不删除文件 但将git中保存的和该文件相关的缓存删除
git rm --cached <filename> 
#4 不删除文件夹 但将git中保存的和该文件夹相关的缓存删除
git rm -r --cached <foldername> 
```
**Remarks:** 命令3、4在将指定文件加入到`.gitignore`作为永久untrack文件时，非常有用。需要注意：已经被git跟踪的文件、文件夹不能通过直接将其名字添加到`.gitignore`文件的方式，在git版本控制中予以忽略。正确的做法是：对于已经被track的文件、文件夹，首先使用命令3、4将git中相关版本信息删除，然后再添加到`.gitignore`文件中，才能禁止git对该文件、文件夹的跟踪。

### Git log
```python
git log  # 只查看当前分支(Head所指的分支)的log情况
git log --oneline   # 简洁的显示版本更新信息
git log -n2  # n2:查看最近两次commit历史
git log -2  # 2:查看最近两次commit历史
git log -n2 --oneline   # 简洁的显示最近两次的版本更新信息
git log branch_name  # 查看指定分支的日志 
​
git log -all  # 列出所有分支的日志
git log --all --graph  # 以图形化的方式查看日志
git log --oneline --all  # 以简洁的方式查看所有分支的日志
git log --oneline --all -n4  # 以简洁的方式查看所有分支的日志
​
git help log  # 以web的方式查看log的帮助文档
git help --web log  # 作用同上 
```

### Branch related
```python
git branch -v  # 查看本地分支的详细情况
git branch -a  # 查看所有分支: 包括远端分支(信息比较简略)
git branch -av  # 查看所有分支情况
 ​
git branch branch_name  # 默认基于当前分支的最后一个commit创建新的分支 
git branch branch_name hash_value  # 基于特定的hashval(commit版本)创建一个新分支 
​
# ???🙉???
git branch -d branch_name
git branch -D branch_name  # 这个分支已经有了一些commit
​
# 切换分支
git checkout branch_name
git checkout master

​# 创建并指向新的分支 git_id=(hashval) or (branchname) or (commitid)
git checkout -b branch_name git_id  
git checkout -b b_name1 9ef147d  # hashval 
git checkout -b b_name2 master  # branchname
 ​
# ???🙉???
git checkout hash_value  # 分离头指针
```

### Git Compare
```python
# 比较两个hashval对应的commit版本 
git diff hash_val1 hash_val2  
# 比较hashval1和hashval2两个版本中的特定两个文件
git diff hash_val1 hash_val2 -- filename1 filename2
​# 对两个分支进行比较
git diff branch_name1 branch_name2  
# 比较两个分支中的特定两个文件
git diff branch_name1 branch_name2 -- filename1 filename2
​
# 当前HEAD分支和前后的分支进行比较
git diff HEAD HEAD^ 
git diff HEAD HEAD^^
git diff HEAD HEAD~
git diff HEAD HEAD~1
git diff HEAD HEAD~2

​# 暂存区和HEAD分支做比较
git diff --cached       
# 暂存区和HEAD分支做比较 只比较特定的两个文件
git diff --cached -- filename1 filename2
# 工作区和暂存区所有文件进行比较​
git diff
# 工作区和暂存区所有文件进行比较
git diff -- filename1 filename2
```

### Commit Changes
```python
# Commit additional staged changes
git commit --amend <CR> 

# ???🙉???
git rebase -i hash_value 
git rabase -i hash_value 
git rebase -i hash_value

# ???🙉???
git rebase origin/master
```

### Reset & Checkout & Restore
```python
# reset
git reset HEAD
git reset HEAD -- file_name1 file_name2
git reset --hard
git reset --hard hash_value

# checkout
git checkout -- file_name
git checkout -- *|.

# restore
```

### Stash
```python
git stash  # 将git文件仓库中的修改藏起来 调用git status不能显示这些更改
git stash list  # 显示藏起来的内容
git stash apply  # 将藏起来的东西拷贝给源文件 但是不销毁藏匿文件 
git stash pop  # 将藏起来的东西剪切给源文件 销毁藏匿文件 
```

### Merge
```python
git merge branch_name1 branch_name2  # 合并两个分支
git merge hash_value1 hash_value2  # 合并两个提交
# ???🙉???
git merge --squash  # 以squash方式进行merge
```

### Git Repo Info 
```python
git cat-file -t|p|s hash_value  # 显示版本库对象的内容、类型及大小信息
git cat-file -t hash_value  # 查看版本库对象的类型
git cat-file -p hash_value  # 查看版本库对象的内容
git cat-file -s hash_value  # 查看版本库对象的大小
```

### Git Remote
```python
git remote add <remotename> <remoteurl> 
git remote -v  # 查看远端仓库连接情况
git remote set-url <remotename> remoteurl  # 修改远端仓库地址
git remote rm <remotename>  # 删除远端仓库
​
git clone <remoteurl>  # clone远端仓库 
git clone --bare <remoteurl>  # 只clone远端仓库的.git目录
​
# 将本地分支推送到远端分支
git push <remotename> <localbranchname> 
# 将本地分支推送到远端分支 并相互关联
git push -u <remotename> <localbranchname>  
# 表示把本地master分支推到远端分支origin/master 并相互关联
git push -u origin master  
# 默认将当前本地分支推到关联的远端分支
git push  
​
# ???🙉???
git fetch <remotename> <localbranchname>
# 将远端分支origin/master fetch到本地
git fetch origin master 
​# 将远端分支fetch到本地 并将远端分支和本地分支合并
git pull <remotename> <localbranchname>  
# ???🙉???
# 以rebase方式进行合并 即将本地分支rebase到远端分支
git pull --rebase 
```

### Git Clean
git clean是一个用来从工作区中移除不想要的文件的命令。可以是编译的临时文件或者合并冲突的文件。

需要谨慎地使用这个命令，因为它被设计为从工作目录中移除untracked文件。git clean过后，worktree中的文件不一定能够找回。移除worktree中的文件更安全的方法是：git stash --all，stash命令将移除的东西放在栈中，在需要时方便找回。 
```python
# 删除untracked file
git clean -f
# 删除untracked file + untracked目录
git clean -fd
# 删除untracked file + untracked目录 + gitignore的untrack文件/目录
# 慎用 一般这个是用来删掉编译出来的.o之类的文件用的
git clean -xfd
# 在用上述git clean前，建议加上-n参数来预览将要删掉的文件 防止重要文件被误删
git clean -nxfd
git clean -nf
git clean -nfd
```

## Reference
> https://www.jianshu.com/p/16080a880740  (关于git4个操作区域和5种编辑状态) <br>
> https://www.cnblogs.com/lsgxeva/p/8540476.html (关于untracked文件进行操作)  <br>
> https://git-scm.com/docs/git-reset (git doc) <br>
> https://stackoverflow.com/questions/572549/difference-between-git-add-a-and-git-add (git add) <br>
> https://www.zhihu.com/question/41667536 (git summary) <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
