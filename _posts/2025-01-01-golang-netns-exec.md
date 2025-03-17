---
layout: post
title: "netns exec utility function (container, golang, linux)"
author: "melon"
date: 2025-01-01 21:44
categories: "2025"
tags:
  - container
  - golang
---

this article covers netns, setns stuff & operation in golang ctx.

<hr>

### # a toy code example for show links in arbitary permitted netns
1 golang env setup script:

```text
#!/bin/sh

proj='netns'  # the project name
image='golang:alpine3.18'
cmd='bash'

script_dir="$(realpath $(dirname "$proj"))"

docker run -it --rm \
    -w /"$proj" \
    -v "$script_dir":/"$proj" \
    -v /var/run/docker.sock:/var/run/docker.sock:rw \
    -v /var/run/netns:/var/run/netns:shared \
    --user "$(id -u)":"$(id -g)" \
    --cap-add=CAP_SYS_ADMIN \
    --cap-add=NET_ADMIN \
    --security-opt apparmor=unconfined \
    --pid=host \
    --privileged \
    --hostname melon \
    $image $cmd

# --pid=host: enable /proc system in container the same as host
# --privileged: enable operations on the container proc system: open & unshare ...

# to enable shared docker in container with host
# -v /var/run/docker.sock:/var/run/docker.sock:rw
# -v /var/run/netns:/var/run/netns:shared
# --cap-add=CAP_SYS_ADMIN \
# --cap-add=NET_ADMIN \
# --security-opt apparmor=unconfined \

# about linux capabilities
# https://man7.org/linux/man-pages/man7/capabilities.7.html
```

2 t.go: main packet of this project.

```text
package main

import (
    "fmt"
    "os"
    "errors"
    "runtime"
    "syscall"
    "github.com/vishvananda/netlink"
    "golang.org/x/sys/unix"
)

func NetNsDo(pid int, do func() error) error {
    resp := make(chan error)
    go func(pid int, do func() error) {
        runtime.LockOSThread()
        defer runtime.UnlockOSThread()

        origPath := fmt.Sprintf("/proc/%d/task/%d/ns/net", os.Getpid(), unix.Gettid())
        origFile, err := os.Open(origPath)
        if err != nil {
            resp <- errors.New("error opening orig net namespace file: " + err.Error())
        }
        defer origFile.Close()

        nsPath := fmt.Sprintf("/proc/%d/ns/net", pid)
        nsFile, err := os.Open(nsPath)
        if err != nil {
            resp <- errors.New("error opening net namespace file: " + err.Error())
        }
        defer nsFile.Close()
        err = unix.Setns(int(nsFile.Fd()), syscall.CLONE_NEWNET)
        if err != nil {
            resp <- errors.New("error setting net namespace: " + err.Error())
        }
        resp <- do()
        unix.Setns(int(origFile.Fd()), syscall.CLONE_NEWNET)
    }(pid, do)
    return <-resp
}

func ListLinks() error {
    links, err := netlink.LinkList()
    if err != nil {
        return errors.New("error listing links: " + err.Error())
    }
    for _, link := range links {
        fmt.Printf("ip links interfaces found: %s\n", link.Attrs().Name)
    }
    return nil
}

func ListVeths() error {
    links, err := netlink.LinkList()
    if err != nil {
        return errors.New("error listing links: " + err.Error())
    }
    for _, link := range links {
        // judge if link is of type netlink.Veth
        if _, ok := link.(*netlink.Veth); ok {
            fmt.Printf("Veth found: %s\n", link.Attrs().Name)
        }
    }
    return nil
}

func main() {
    // inspect pid 1 inside container to get the interfaces outside
    pid := 1
    err := NetNsDo(pid, ListLinks)
    if err != nil {
        fmt.Printf("Error: %s\n", err)
    }

    fmt.Println("--------------------------------")

    err = NetNsDo(pid, ListVeths)
    if err != nil {
        fmt.Printf("Error: %s\n", err)
    }
}
```

3 makefile: project insight utiliy script.

```text
DEPS =  github.com/vishvananda/netlink \
    github.com/vishvananda/netns \
    golang.org/x/sys/unix

goroot = $(addprefix ../../../,$(1))
unroot = $(subst ../../../,,$(1))
fmt = $(addprefix fmt-,$(1))

BIN_NAME := list_link_in_netns
VERSION ?= $(shell git rev-parse --is-inside-work-tree 2>/dev/null && git describe --tags --abbrev=0 | \
    sed 's/^v//' || echo "v1.0")

BUILD_DATE := $(shell date '+%Y-%m-%d-%H:%M:%S')

LDFLAGS := version=${VERSION}
LDFLAGS := ${LDFLAGS} -X build_date=${BUILD_DATE}

default: build

# setup golang project dep pkg
$(call goroot,$(DEPS)):
    go get $(call unroot,$@)

# setup go project module info
modinit:
    -go mod init melon
    -go mod tidy

deps: modinit $(call goroot,$(DEPS))  ## setup golang project deps
    @echo "setup golang deps..."

format: ## check coding style
    @echo "reformating using gofmt..."
    @DIFF=$$(gofmt -d .); echo -n "$$DIFF"; test -z "$$DIFF"

build:  ## compile the golang project
    @echo "building ${BIN_NAME} ${VERSION}"
    @echo "GOPATH=${GOPATH}"
    rm -rf bin/*; CGO_ENABLED=0 go build -ldflags "${LDFLAGS}" -o bin/${BIN_NAME}

clean:  ## cleanup
    @test ! -e bin/${BIN_NAME} || rm bin/${BIN_NAME}
    @test ! -e go.mod || rm go.mod
    @test ! -e go.sum || rm go.sum

test:   ## run go test
    sudo bin/${BIN_NAME}

help:   ## display this help
    @grep -h -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; \
    {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'
```

4 test the main script functionality via makefile.

```text
melon:/netns$ make help
format                         check coding style
deps                           setup golang project deps
build                          compile the golang project
clean                          cleanup
test                           run go test
help                           display this help screen
```

```text
melon:/netns$ make deps
go mod init melon
go: creating new go.mod: module melon
go: to add module requirements and sums:
        go mod tidy
go mod tidy
go: finding module for package golang.org/x/sys/unix
go: finding module for package github.com/vishvananda/netlink

go: toolchain upgrade needed to resolve golang.org/x/sys/unix
go: golang.org/x/sys@v0.31.0 requires go >= 1.23.0 (running go 1.22.1; GOTOOLCHAIN=local)
make: [makefile:26: modinit] Error 1 (ignored)
go get github.com/vishvananda/netlink
go: added github.com/vishvananda/netlink v1.3.0
go: added github.com/vishvananda/netns v0.0.4
go: added golang.org/x/sys v0.10.0
go get github.com/vishvananda/netns
go: upgraded github.com/vishvananda/netns v0.0.4 => v0.0.5
go get golang.org/x/sys/unix
go: golang.org/x/sys/unix: golang.org/x/sys@v0.31.0 requires go >= 1.23.0 (running go 1.22.1; GOTOOLCHAIN=local)
make: *** [makefile:21: ../../../golang.org/x/sys/unix] Error 1
```

```text
melon:~$ make build
building list_link_in_netns v1.0
GOPATH=/go
rm -rf bin/*; CGO_ENABLED=0 go build -ldflags "version=v1.0 -X build_date=2025-03-10-09:10:59" -o bin/list_link_in_netns
```

```text
melon:~$ make test
sudo bin/list_link_in_netns
ip links interfaces found: lo
ip links interfaces found: veth01fc38a
ip links interfaces found: dummy0
ip links interfaces found: eth0
ip links interfaces found: bond0
ip links interfaces found: virbr0
ip links interfaces found: virbr0-nic
ip links interfaces found: docker0
ip links interfaces found: br-37105304fb02
ip links interfaces found: veth9f0eec5
ip links interfaces found: veth18f950c
ip links interfaces found: veth4d623d5
ip links interfaces found: veth34efe8a
ip links interfaces found: vetha3ee161
ip links interfaces found: veth98dfcea
ip links interfaces found: veth084647a
ip links interfaces found: vethd71fea9
```

```text
melon:~$ make clean
```

<hr>

### # exec do in certain pid's net, mnt, pid namespace
the following code exhibit a basic things for:  
1 why we need lock the os.thread before switch ns for net, mnt, pid.  
2 why mnt is a little bit different among other ns like net, pid when programming.  
3 why we dont need the unlock os thread at end.

```text
func NamespaceDo(pid int, do func() error) error {
    // helper function to set pid fd into namespace specified by /proc/${pid}/ns/${namespace_str}
    setns := func(pid int, namespaceType string) error {
        nsPath := fmt.Sprintf("/proc/%d/ns/%s", pid, namespaceType)
        f, err := os.Open(nsPath)
        if err != nil {
            return fmt.Errorf("failed to open namespace %s: %w", nsPath, err)
        }
        defer f.Close()
        if err := unix.Setns(int(f.Fd()), 0); err != nil {
            return fmt.Errorf("failed to setns for %s: %w", namespaceType, err)
        }
        return nil
    }

    // worker function
    resp := make(chan error)
    go func(pid int, do func() error) {
        runtime.LockOSThread()
        if err := unix.Unshare(unix.CLONE_FS); err != nil {
            resp <- fmt.Errorf("error unsharing file system attributes: %s", err)
            return
        }
        namespaces := []string{"net", "pid", "mnt"}
        for _, ns := range namespaces {
            if err := setns(pid, ns); err != nil {
                resp <- fmt.Errorf("error setting namespace %s: %v", ns, err)
                return
            }
        }
        // entered in the mnt and net namespace associated with the pid
        log.Debugf("Enter pid %d's namespaces", pid)
        resp <- do()
    }(pid, do)

    return <-resp
}
```

$ 1 why we need to bind the goroutine to certain os thread?  
bind the goroutine on the os thread its firstly running on, avoid dispatch to other thread in the middle
of this goroutine, which is unreasonable in this set namespace & execute.

go runtime use a work-stealing scheduler, which maintain a pool of thread and schedule goroutine onto them.
goroutine are multiplexed onto a smaller num of os thread, allowing tons of goroutine to run concurrently
on a limited num of thread.

go scheduler can preempt goroutines: if a goroutine blocks or io, or channel operations... the scheduler
will switch to runnable goroutine.
if a os thread has no goroutine to exec, it can steal goroutine from other threads's que, to balance loads.

$ 2 why need unshare resource from parent ctx?  
the is needed for setns (mnt), to separate shared resource from parent proc, in this case to avoid
polluting the parent mnt in onwards goroutine operations.
clone_fs flag specifically unshare filesystem attributes, including:
root dir (chroot), current workdir, umask value.

$ 3 the sequence of []string{"net", "pid", "mnt"} for handling matters  
mnt namespace should be last to deal with, because the mnt namespace affect the `/proc/${pid}/ns` viewpoint.
if set pid to new mnt firstly, following setns for net/pid ns will lost it's real target.

$ 4 why does golang does not possess a single fork?  
for pid namespace, only the child precess will be put into the target pid ns to execute do(),
so it's better to run do() after "fork" in child process to activate the associated pid namespace.
but golang does not provide a single "fork" because golang's runtime requires "exec" right after the "fork".

$ 5 why not using defer runtime.UnlockOSThread()?  
because, we want to make sure the os thread is terminated right after goroutine end.
if with it, the go runtime will detach goroutine from os thread whenever it end, so the osthread used is
available to steal goroutine from other busy thread, or runtime might dispatch other goroutine on this osthread,
finally leading to namespace pollution.
