---
layout: post
title: "carriage return in shell variable (shell)"
author: "melon"
date: 2024-01-03 21:10
categories: "2024"
tags:
  - shell
---

{% raw %}
the first script is showing a weird output:
the cmd1 buildup using the return value from nested container cmd is somehow
overlapped by its own ending chars.
```text
#!/bin/sh

pid1=$(docker exec -it setup_shelf_boards_1_0z1u5p_metung \
       bash -c "docker exec -it olt_1 \
       bash -c \" docker inspect --format '{{.State.Pid}}' lt_1 \" ")
echo $pid1                                                # 5971 (with carriage return)

cmd1="docker exec -it olt_1 readlink /proc/$pid1/ns/pid"
echo $cmd1                                                # /ns/pidexec -it olt_1 readlink /proc/5971
```

the second script is showing the desired output:
simply get rid of the carriage return in pid2, then echo the concatenated string out.
this time we can derive the expected output:
```text
#!/bin/sh

pid2=$(docker exec -it setup_shelf_boards_1_0z1u5p_metung \
       bash -c "docker exec -it olt_1 \
       bash -c \"docker inspect --format '{{.State.Pid}}' lt_1\"" | tr -d '\r')
echo $pid2                                                # 5971 (without carriage return)

cmd2="docker exec -it olt_1 readlink /proc/$pid2/ns/pid"
echo $cmd2                                                # docker exec -it olt_1 readlink /proc/5971/ns/pid
```
{% endraw %}

as a conclusion, the carriage return can be stored in shell variable, and once the variable
got interpreted by shell, then it will make the output disordered.

moreover, there's no difference between the output of echo $pid1 and echo $pid2, even
pid1 contains carriage return.

