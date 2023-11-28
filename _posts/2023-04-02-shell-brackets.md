---
layout: post
title: "brackets usages (shell)"
author: "twistfatezz"
date: 2023-04-02 09:48
categories: "2023"
tags:
  - shell
---

### # ${var}
work as the reference to the variable.
```shell
var="hello world"
echo ${var}
```
```txt
hello world
```

<hr>

### # $(cmd)
get cmd stdout, output nothing when cmd failed, the same with `` `cmd` ``.
```shell
$ quota -vs -u $(id -u)
Disk quotas for user metung (uid 3002007):
Filesystem               space   quota   limit   grace   files   quota   limit   grace
FAS2020A:/vol/home/data  116K    2500M   2510M           31      4295m   4295m
FAS3210A:/vol/repo1      0K      18432M  20480M          0       4295m   4295m
FAS3210A:/vol/repo2      0K      18432M  20480M          0       4295m   4295m
FAS3210A:/vol/repo3      0K      18432M  20480M          0       4295m   4295m
```
self add operation using the format of: $() + bc, which support all type computation (integer/float/double):
```shell
i=0; i=$(echo "$i+0.5" | bc)
echo $i
```
```txt
0.5
```

<hr>


### # `` `cmd` ``
get cmd stdout, output nothing when cmd failed, the same with $(cmd)
```shell
$ quota -vs -u `id -u`
Disk quotas for user metung (uid 3002007):
     Filesystem   space   quota   limit   grace   files   quota   limit   grace
FAS2020A:/vol/home/data
                   116K   2500M   2510M              31   4295m   4295m
FAS3210A:/vol/repo1
                     0K  18432M  20480M               0   4295m   4295m
FAS3210A:/vol/repo2
                     0K  18432M  20480M               0   4295m   4295m
FAS3210A:/vol/repo3
                     0K  18432M  20480M               0   4295m   4295m
```

<hr>


### # `$((expression))`
get expression numeric value, expression can be in the form of c, the same with `` `expr expression` ``.
```shell
for ((;;)); do
    sleep 1s
    i=$((i+1))  # self-add
    if [[ i -eq 5 ]]; then
        break
    fi
    echo "press ctrl+c to interrupt this infinite loop"
done
```
```txt
press ctrl+c to interrupt this infinite loop
press ctrl+c to interrupt this infinite loop
press ctrl+c to interrupt this infinite loop
press ctrl+c to interrupt this infinite loop
press ctrl+c to interrupt this infinite loop
```
```shell
echo $((3+2))         # 5  numeric operation
echo $((3>2))         # 1  boolean operation
echo $((25<3 ? 2:3))  # 3  triplet operation
echo $((var=2+3))     # 5
echo $var             # 5
echo $((var++))       # 5  post self add
echo $var             # 6
echo $((++var))       # 7  pre self add
echo $var             # 7
```
self add operation, note: $(()) only supports integer computation:
```shell
i=0; i=$((i+1)); echo $i
i=$((i+0.5)); echo $i
```
```txt
1
./1.sh: line 2: i+0.5: syntax error: invalid arithmetic operator (error token is ".5")
```

<hr>


### # `` `expr expression` ``
print the value of expression to stdout, see expr \-\-help for details.
```shell
for ((;;)); do
    sleep 1s
    i=`expr $i+1`  # self-add
    if [[ i -eq 5 ]]; then
        break
    fi
    echo "press ctrl+c to interrupt this self-add loop"
done
```
```txt
press ctrl+c to interrupt this self-add loop
press ctrl+c to interrupt this self-add loop
press ctrl+c to interrupt this self-add loop
press ctrl+c to interrupt this self-add loop
press ctrl+c to interrupt this self-add loop
```
expr used in for loop: enable dynamic range in it.
```shell
var=4
for i in $(seq 3 $(expr 3 + $var)); do
    echo $i
done
```
```txt
3
4
5
6
7
```

<hr>

### # `()` - (cmd1; cmd2; cmd3;)
create a subshell, run cmd in subshell sequentialy.
```shell
var=bob                 # set parent shell var
(var=jack; echo $var)   # open a subshell && run cmd in it, output jack
echo $var               # current shell var is not changed, output bob
```

<hr>

### # `{}` - { cmd1; cmd2; cmd3;}
run cmd in current shell sequentially.
```shell
var=bob                  # set parent shell var
{ var=jack; echo $var;}  # run cmd in cur shell, output jack
echo $var                # parent shell var is changed, output jack
```
note: the space between left `{` and cmd1 is mandantory:
```shell
var=bob
{var=jack; echo $var;}
echo $var
```
```txt
./1.sh: line 2: syntax error near unexpected token `}'
./1.sh: line 2: `{var=jack; echo $var;}'
```

<hr>

### # `(())`
enable +=*/ in shell, compute expression inside (()).
```shell
a=1; b=2; c=3;
((a=a+1)); echo $a;                    # changing a's value
a=$((a+1, b++, c++));  echo $a,$b,$c;  # the expression value is assigned to $a
```
```txt
2
3 3 4
```
compute the expr inside (()), value of last cmd in (()) is expression value. 
```shell
a=1; b=2; c=3;
((a=a+1)); echo $a;
d=$((a+1, b++, c++));  echo $a,$b,$c,$d;
```
```txt
2
3 3 4 3
```
mimicing "if condition" in 1 line:
```shell
a=1;
echo $((a>1? 1:0))                                # only numeric value supported
((a>2)) && echo "this statement is not printed"   # no output
((a<2)) && echo "this statement is printed"
```
```txt
1

this statement is printed
```
used in if/while/for statements:
```shell
num=10; total=0;
for((i=0; i<=num; i++)); do
    ((total+=i));
done
echo $total;
```
```txt
50
```
```shell
num=10; total=0; i=0;
while((i<=num)); do
    ((total+=i,i++));
done
echo $total;
```
```txt
50
```
```shell
total=10000;
if((total>=5050)); then
    echo "ok";
fi
```
```txt
ok
```

<hr>

### # `${var:} / ${var%} / ${var#}`: variable replacement and match operations
replacement:
```shell
${var:-string}   # if var=null, then `${var:-string}`=string
${var:=string}   # if var=null, then var=string
${var:+string}   # if var!=null, then var=string
```
match:
```shell
${var%pattern}   # match from the right end, get the 1st match
${var%%pattern}  # match from the right end, get the 2nd match
${var#pattern}   # match from the left end, get the 1st match
${var##pattern}  # match from the left end, get the 2st match
```



> ref: https://www.cnblogs.com/fnlingnzb-learner/p/10020697.html  
> ref: https://www.runoob.com/w3cnote/linux-shell-brackets-features.html
