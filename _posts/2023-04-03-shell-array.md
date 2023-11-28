---
layout: post
title: "array (shell)"
author: "twistfatezz"
date: 2023-04-03 20:53
categories: "2023"
tags:
  - shell
---

### # basic array operations
```shell
# array definition
arr=(1 2 "donkey" a)

# return the whole arr
echo "the whole arr is: ${myarr[@]}"

# get i-th element of arr 
echo "the second member is: ${myarr[2]}"

# get the length of arr
echo "the length of the arr is: ${#myarr[@]}"

# return arr slicing
#idx: -6 -5 -4  -3    -2     -1
#idx:  0  1  2   3     4      5
arr=(1 2 3 "john" "kitty" 5)
brr=("${arr[@]:1:2}")   # slice idx range: [1,1+2)
crr=("${arr[@]:1}")     # slice idx range: [1,end)
echo "${brr[@]}"        # 2 3
echo "${brr[1]}"        # 3
echo "${crr[@]}"        # 2 3 john kitty 5
echo "${crr[@]: -2:2}"  # slice start=-2, length=2, output: kitty 5
```

<hr>

### # get the array content/element by arr string name
```shell
# ref: https://stackoverflow.com/a/61267361
function get {
    # get arr content by string name
    arr_name=$1
    arr=$arr_name[@]
    arr=("${!arr}")
    arr_content="${arr[@]}"
    echo $arr_content

    # get element by string name
    arr_ele="${arr[2]}"
    echo $arr_ele
}
a=(1 2 3 4 5); get a;

# output:
# 1 2 3 4 5
# 3
```

<hr>

### # judge the existence of an element in array
(1) using shell build-in regex
```shell
# ref: https://askubuntu.com/a/995110
# 1: array arguments of function should be the LAST ONE of argument list
# 2: only one array arguments can be passed, the 2nd 3rd array argument will be piled to one
function isin {
    value=$1
    shift     # shift all arguments to the left($1 is flushed, only array left)
    array=$@
    # if in
    if [[ "${array[*]}" =~ "${value}" ]]; then
        echo "in"  # return 0
    fi
    # if not in
    if [[ ! "${array[*]}" =~ "${value}" ]]; then
        echo "not in"  # return 1
    fi
}

arr=(1 2 3 4 "john"); item=john;
isin $item "${arr[@]}"
```

(2) using build-in character substitution
```shell
# sample-1
function isin {
    arr=$1; item=$2;  # params
    arr="${arr[@]}"   # arr content
    if [[ ${arr/$item/} != ${arr} ]]; then
        echo "in"
    else
        echo "not in"
    fi
}

arr=(1 2 3 4 "john"); item1="john"; item2="7"
isin arr $item1   # output: in
isin arr $item2   # output: not in
```
```shell
# sample-2
function judge {
    a=(3 5 8 10 6)
    b=(6 5 4 12)
    c=(14 7 5 7)
    ele=7
    for ele in ${a[@]}; do
        # string substitution: element exist in array a/b simultaneously
        if [[ ${b[@]/$ele/} != ${b[@]} && ${c[@]/$ele/} != ${c[@]} ]]; then
            echo $ele
        fi
    done
}
```

(3) using the grep tool 
```shell
function judge {
    a=(3 5 8 10 6)
    b=(6 5 4 12)
    c=(14 7 5 7)
    ele=7
    for ele in ${a[@]}; do
        # grep -wq: element exist in array a/b simultaneously
        if echo "${b[@]}" | grep -wq "$ele" && echo "${c[@]}" | grep -wq "$ele"; then
            echo $ele
        fi
    done
}
```

