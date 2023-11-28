---
layout: post
title: "reformat nvim paste input (shell)"
author: "melon"
date: 2023-06-20 20:09
categories: "2023"
tags:
  - shell
---

todo: this script could be used with xclip/xsel commandline tools to enable processing automation
      the in/out could be pipe with xclip/xsel stdin/stdout

### # script for macos with pbcopy/pbpaste support
```shell
#!/bin/sh
# input: system clipboard content
# output: system clipboard output

content=$(pbpaste)

if [[ -z $content ]]; then
    echo "system clipboard empty!"
else
    IFS=''
    echo $content | while read -r line; do
        pattern="[^a-zA-Z{}#\/\/]"                        # rational chars are: '{',  '}', '#', '//', 'a-z/A-Z'
        count=0                                           # count the number of spaces for each line's indentation

        for i in {0..49}; do
            if [[ ${line:$i:1} =~ $pattern ]]; then       # if the i-th char matched the pattern predefined
                count=$((count+1))
            else
                break
            fi
        done
        
        space=" "                                         # padding space string to mimic indentation of each line
        str="${space}"
        while (( ${#str} < $count )); do
            str="${str}${str}"
        done
        str="${str:0:$count}"
        newline=$(echo $line | sed "s/^$pattern*/$str/g")
        echo $newline >> tmp 
    done
    
    sed 's/ *$//' tmp | pbcopy                            # get rid of all trailing spaces
    rm tmp
    echo "reformated text pasted!"
fi
```

<hr>

### # a sample from system clipboard from nvim editor
the paste result is polluted by line number and indentation lines.
```txt
  1 #!/bin/bash
  2 
  3 # use random container name to avoid naming conflicts     
  4 
  5 # edit $1: input file for edit
  6 # edit current dir: if no params provided
  7 if [ $# -eq 0 ]; then
  8     dir="${PWD}"
  9     docker run -it --rm -v ${dir}:/root/mount/ nvim:v1 /root/mount/
 10 else
 11     dir="${PWD}/${1}"
 12     if [[ ${1} == .* ]]; then
 13         docker run -it --rm -v ${dir}:/root/mount/${1} nvim:v1 ${1}
 13 │   while (( ${#str} < $count )); do
 12 │   │   str="${str}${str}"
 11 │   done
 10 │   str="${str:0:$count}"
  9 │   newline=$(echo $line | sed "s/^$pattern*/$str/g")
  8 │   # remove all-space line as \
  7 │   # newline=$(echo $newline | sed "s/^\s*$//g")
  6 │   echo $newline >> tmp
  5 done < $1
  9 │   newline=$(echo $line | sed "s/^$pattern*/$str/g")
  8 │   // remove all-space line as \
  7 │   // newline=$(echo $newline | sed "s/^\s*$//g")
  8 │   // remove all-space line as \
  8 │   // remove all-space line as \
  8 │   // remove all-space line as \
  7 │   // newline=$(echo $newline | sed "s/^\s*$//g")
  6 │   echo $newline >> tmp
  5 done < $1
```

<hr>

### # reformatted output
the script replace all unwanted prefix of each line, and redirect the reformatted results into system clipboard with the orginal indentation keeped.
```txt
    #!/bin/bash

    # use random container name to avoid naming conflicts

    # edit $1: input file for edit
    # edit current dir: if no params provided
    if [ $# -eq 0 ]; then
        dir="${PWD}"
        docker run -it --rm -v ${dir}:/root/mount/ nvim:v1 /root/mount/
    else
        dir="${PWD}/${1}"
        if [[ ${1} == .* ]]; then
            docker run -it --rm -v ${dir}:/root/mount/${1} nvim:v1 ${1}
        while (( ${#str} < $count )); do
            str="${str}${str}"
        done
        str="${str:0:$count}"
        newline=$(echo $line | sed "s/^$pattern*/$str/g")
        # remove all-space line as \
        # newline=$(echo $newline | sed "s/^\s*$//g")
        echo $newline >> tmp
    done < $1
        newline=$(echo $line | sed "s/^$pattern*/$str/g")
        // remove all-space line as \
        // newline=$(echo $newline | sed "s/^\s*$//g")
        // remove all-space line as \
        // remove all-space line as \
        // remove all-space line as \
        // newline=$(echo $newline | sed "s/^\s*$//g")
        echo $newline >> tmp
    done < $1
```

<hr>

### # embed the script into nvim shortcut (todo)
