---
layout: post
title: "dockerfile ENV & ARG usages (docker & dockerfile)"
author: "melon"
date: 2023-10-16 20:16
categories: "2023"
tags:
  - container
---

### # introduction

<hr>

### # set docker ENV variable
given the directory tree as follows:
```text
.
├── Dockerfile
├── entrypoint.sh
└── envfile
```
dockerfile for building the test image:
```text
FROM alpine:latest

ENV MY_ENV_VAR=world                             # setup ENV variable in dockerfile
ADD entrypoint.sh /
ENTRYPOINT /entrypoint.sh "Hello, $MY_ENV_VAR!"  # call entrypoint.sh when container startup
```
entrypoint.sh is used for print the ENV variable "$MY_ENV_VAR" when executed:
```text
#!/bin/sh
echo $1                                          # print out the first param passed
```
envfile: a file that hold key-value pairs of env variable, as follows:
```text
MY_ENV_VAR=-smbios type=1,product="TIMOS: chassis=IXR-R6 card=cpiom-ixr-r6 slot='A'"
ENV_VAR2=world
```
build the docker image defined above and test with different configurations by:
```text
$ docker build -t testenv .                                   # build

$ docker run --rm testenv:latest                              # print out variable defined in dockerfile
hello, world!

$ docker run --rm -e MY_ENV_VAR=worlddddddd testenv:latest    # the cmdline -e will override ENV in dockerfile
hello, worlddddddd

$ docker run --rm --env-file envfile testenv:latest           # the cmdline --env-file will override ENV in dockerfile
hello, -smbios type=1,product="TIMOS: chassis=IXR-R6 card=cpiom-ixr-r6 slot='A'"
```
using '--env-file', multiple key-value pairs can be transfer to entrypoint.sh concisely, in which the value can contain some special characters (single/double quotes).
If the envfile is located at the same dir as docker run command, no explicit path needed.

<hr>

### # ENV vs ARG
the lifetime of ENV and ARG are different from the docker image to container lifetime:
```txt
                             ┌───────────────────────────────────┐
                             │ ┌──────────┐   ARGs are available │
                             │ │dockerfile│                      │
                             │ └────┬─────┘                      │
                         ┌───┼──────│────────────────────────────┼───┐
                         │   │      │<--- building image         │<--┼--- intersection part
                         │   │      │                            │   │
                         │   └──────│────────────────────────────┘   │
                         │       ┌──┴──┐                             │
                         │       │image│                             │
                         │       └──┬──┘                             │
                         │          │<--- docker run                 │
                         │     ┌────┴────┐                           │
                         │     │container│                           │
                         │     └─────────┘        ENVs are available │
                         └───────────────────────────────────────────┘
```
to find ARG / ENV / build cmd of image, try using:
```text
$ docker history --no-trunc ${image_id}
```
to find ENV of image, can also check with:
```text
$ docker inspect ${image_id} | grep ENV
```

<hr>

### # ARG & ENV for sensitive data
setting ARG and ENV values leaves traces in the Docker image, so don’t use them for secrets which are not meant to stick around.  
Even though ARG values are not available to the container, they can easily be inspected through docker CLI.

ARG and ENV are a poor choice for sensitive data if untrusted users have access to your images.  
Multi-stage builds (see shuttle/desc/dockermultistagebuild) is good method to hide info by storing them in previous base image.

\>>> dockerfile test case 1: the ENV & ARG variable lifetime.
```text
$ cat ./Dockerfile.v1
FROM alpine:latest 
ENV ALPINE_VER=3.17
ARG SITE=artifactory-blr1.int.net.xxxxx.com
ARG REBORN_BUILD_KEY=reborn-638089aa
ARG ARTIFACTORY_REBORN=https://$SITE/artifactory/rebornlinux-generic-local

$ docker build -f ./Dockerfile.v1 -t test:v1 .
$ docker history --no-trunc test:v1                                           # trunc output manually
IMAGE           CREATED          CREATED BY                                                         SIZE      COMMENT
sha256:a4e6af   36 seconds ago   /bin/sh -c #(nop)  ARG ARTIFACTORY_REBORN=https://artifactory...   0B
sha256:bf5c00   37 seconds ago   /bin/sh -c #(nop)  ARG REBORN_BUILD_KEY=reborn-638089aa            0B
sha256:4e9072   37 seconds ago   /bin/sh -c #(nop)  ARG SITE=artifactory-blr1.int.net.xxxxx.com     0B
sha256:fab063   39 seconds ago   /bin/sh -c #(nop)  ENV ALPINE_VER=3.17                             0B
sha256:7e01a0   2 months ago     /bin/sh -c #(nop)  CMD ["/bin/sh"]                                 0B
<missing>       2 months ago     /bin/sh -c #(nop)  ADD file:32ff5e in /                            7.34MB

$ docker run -it --rm test:v1 sh                                              # access the container of test:v1
/ # echo $ALPINE_VER                                                          # the ENV is still available
3.17
/ # echo $ARTIFACTORY_REBORN                                                  # null: the ARG is not available

/ # echo $SITE                                                                # null: the ARG is not available
```

\>>> hide the ARG from container user: define ARG above FROM clause, redecla  re it later.
```text
$ cat ./Dockerfile.v2                                                         # dockerfile undertest v2
ARG ALPINE_VER=3.17                                                           # define variable=value
ARG SITE=artifactory-blr1.int.net.xxxxx.com
ARG REBORN_BUILD_KEY=reborn-638089aa

FROM alpine:latest                                                            # from

ARG ALPINE_VER                                                                # declare variable again
ARG SITE
ARG REBORN_BUILD_KEY
ARG ARTIFACTORY_REBORN=https://$SITE/artifactory/rebornlinux-generic-local

$ docker build -f ./Dockerfile.v2 -t test:v2 .                                # build image

$ docker history --no-trunc test:v2                                           # ARG name visible, value are hidden
IMAGE            CREATED          CREATED BY         SIZE      COMMENT
sha256:a9e61d7   11 seconds ago   /bin/sh -c #(nop)  ARG ARTIFACTORY_REBORN=https://artifactory-blr1...   0B
sha256:6b0f10c   11 seconds ago   /bin/sh -c #(nop)  ARG REBORN_BUILD_KEY                                 0B
sha256:7a2272e   11 seconds ago   /bin/sh -c #(nop)  ARG SITE                                             0B
sha256:49a0965   12 seconds ago   /bin/sh -c #(nop)  ARG ALPINE_VER                                       0B
sha256:7e01a0d   4 weeks ago      /bin/sh -c #(nop)  CMD ["/bin/sh"]                                      0B
<missing>        4 weeks ago      /bin/sh -c #(nop)  ADD file:32ff5e7a78b890996ee4681cc0a26... in /       7.34MB
```

\>>> failed to hide ENV: the ENV cannot be defined above FROM clause.
```text
$ cat ./dockerfile.v3
# ENV ALPINE_VER=3.17                                                         # wrong: Error response from daemon: 
                                                                              #        no build stage in current context
                                                                              #        cannot use ENV before FROM clause
ARG ALPINE_VER=3.17                                                           # define ALPINE_VER
ARG SITE=artifactory-blr1.int.net.xxxxx.com
ARG REBORN_BUILD_KEY=reborn-638089aa

FROM alpine:latest                                                            # image base

# ENV ALPINE_VER                                                              # wrong!: must follow ENV xxx=xxx format
ARG ALPINE_VER                                                                # declare reuse of ALPINE_VER (3.17)
ENV ALPINE_VER=$ALPINE_VER                                                    # setup ENV using ARG declared
ARG SITE
ARG REBORN_BUILD_KEY
ARG ARTIFACTORY_REBORN=https://$SITE/artifactory/rebornlinux-generic-local

$ docker build -f ./Dockerfile.v3 -t test:v3 .                                # build image

$ docker history test:v3 --no-trunc                                           # check for the ENV/ARG hidden effects
IMAGE           CREATED         CREATED BY                                                    SIZE      COMMENT
sha256:9fb757   2 minutes ago   /bin/sh -c #(nop)  ARG ARTIFACTORY_REBORN=https://artifa...   0B
sha256:b57bc6   2 minutes ago   /bin/sh -c #(nop)  ARG REBORN_BUILD_KEY                       0B
sha256:de1a99   2 minutes ago   /bin/sh -c #(nop)  ARG SITE                                   0B
sha256:f13213   2 minutes ago   /bin/sh -c #(nop)  ENV ALPINE_VER=3.17                        0B
sha256:49a096   5 weeks ago     /bin/sh -c #(nop)  ARG ALPINE_VER                             0B
sha256:7e01a0   2 months ago    /bin/sh -c #(nop)  CMD ["/bin/sh"]                            0B
<missing>       2 months ago    /bin/sh -c #(nop)  ADD file:32ff5e7a78b890996ee468 in /       7.34MB

$ docker run -it --rm test:v3 sh                                              # access the container
/ # echo $ALPINE_VER                                                          # the ENV variable is still accessible
3.17
/ # echo $SITE                                                                # the ARG variable is not accessible

/ # echo $ARTIFACTORY_REBORN
```

\>>> inheriting the ENV from "parent image":
```text
$ cat dockerfile.v4                                                           # a single line dockerfile
FROM test:v2                                                                  # this image is a "child" image of test:v2

$ docker build -f ./dockerfile.v4 -t test:v4 .                                # buildup

$ docker history test:v4                                                      # check ARG/ENV visibility
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
ca1e454e753b   55 seconds ago   /bin/sh -c #(nop)  ARG TEST                     0B
9fb7576fdec7   24 minutes ago   /bin/sh -c #(nop)  ARG ARTIFACTORY_REBORN=ht…   0B
b57bc688c450   24 minutes ago   /bin/sh -c #(nop)  ARG REBORN_BUILD_KEY         0B
de1a99d0ce3e   24 minutes ago   /bin/sh -c #(nop)  ARG SITE                     0B
f1321325e93a   24 minutes ago   /bin/sh -c #(nop)  ENV ALPINE_VER=3.17          0B
49a09657211f   5 weeks ago      /bin/sh -c #(nop)  ARG ALPINE_VER               0B
7e01a0d0a1dc   2 months ago     /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>      2 months ago     /bin/sh -c #(nop) ADD file:32ff5e7a78b890996…   7.34MB

$ docker run -it --rm test:v4 sh                                              # access the container 
/ # echo $ALPINE_VER                                                          # the ENV can be inherited & accessible
3.17
/ # echo $SITE                                                                # the ARG cannot be accessible

/ # exit
```

<hr>

### # reference
https://docs.docker.com/engine/reference/commandline/run/#env
