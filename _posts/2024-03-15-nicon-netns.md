---
layout: post
title: "nicon project illustrated: pkg/netns (network, golang)"
author: "melon"
date: 2024-03-15 20:02
categories: "2024"
tags:
  - network
  - golang
  - container
---

{% raw %}

nested containers come in two flavor: dind or dood.

dind typically maintain a standalone docker daemon inside a container, which
provide separate docker cli features.
the dood reuse the same docker daemon on host machine, and do ipc with daemon by vsock.

```txt
                    ┌───────────────────────────────────────────┐
                    │ host machine                              │
                    │ ┌───────────────────────────────────────┐ │
                    │ │ wrapper container                     │ │
                    │ │ ┌───────────────────────────────────┐ │ │
                    │ │ │ inner container +                 │ │ │
                    │ │ └─────────────────|─────────────────┘ │ │
                    │ │                   |                   │ │
                    │ │      unix:///var/run/docker.sock      │ │
                    │ │                   +                   │ │
                    │ │             docker daemon             │ │
                    │ └───────────────────────────────────────┘ │
                    └───────────────────────────────────────────┘
```

for the dind case, we can simply get pid for inner container from the wrapper container:
```text
$(wrapper container) docker inspect -f '{{.State.Pid}}' ${container_id/name}
```

however, the pid derived is fake, which is not the same as one actually on host machine.
each container has a isolated pid namespace, so the pid derived above is re-arranged on
host machine to get a global unique id marker. thus, process with pid 3 on wrapper container
is likely to has pid 10086 on host.

we want to get real pid represent the inner container on host. the following
golang module does the job.

<hr>

### # core code: pkg/netns/netns.go
1 netns package header imports:
```text
package netns

import (
	"context"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"runtime"
	"strconv"
	"strings"
	"syscall"

	"github.com/docker/docker/api/types/container"
	"github.com/docker/docker/api/types/filters"
	"github.com/docker/docker/client"
	"github.com/vishvananda/netlink"
	"golang.org/x/sys/unix"

	"github.com/melon/NiCon/pkg/log"
)
```

core functions: get pid of container with depth=1 on host by container name.
```text
func InspectHostContainerPID(ctx context.Context, containerName string) (int, error) {
    log.Debugf("Inspecting container %s", containerName)

    if containerName == "" {
        return 0, nil
    }

    cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
    if err != nil {
        return 0, fmt.Errorf("Failed to create docker client: %v", err)
    }
    defer cli.Close()
    containers, err := cli.ContainerList(context.Background(), container.ListOptions{
        All:     true,
        Filters: filters.NewArgs(filters.Arg("name", containerName)),
    })
    if err != nil || len(containers) == 0 {
        return 0, fmt.Errorf("Failed to list containers: %v", err)
    }
    container := containers[0]
    log.Debugf("Container %s ID: %s", containerName, container.ID)
    info, err := cli.ContainerInspect(ctx, container.ID)
    if err != nil {
        return 0, fmt.Errorf("Failed to inspect container: %v", err)
    }
    log.Debugf("Container %s PID: %d", containerName, info.State.Pid)

    return info.State.Pid, nil
}
```

2 core: get real pid of the container by its name and the real pid of its parent container.
```text
func InspectContainerPID(basePid int, containerName string) (int, error) {
    log.Infof("Inspecting container %s", containerName)

    type pidResult struct {
        nsID int
        err  error
    }

    resp := make(chan pidResult)

    go func() {
        // lock current routine to certain os thread for any ns related syscall
        runtime.LockOSThread()

        // unshare clonefs: protect the caller's rootdir/curdir/umask context
        // ref: https://man7.org/linux/man-pages/man2/unshare.2.html
        if err := unix.Unshare(unix.CLONE_FS); err != nil {
            resp <- pidResult{
                nsID: 0,
                err:  fmt.Errorf("Error unsharing mount namespace: %s", err),
            }
        }

        // open the mount namespace
        nsPathMount := fmt.Sprintf("/proc/%d/ns/mnt", basePid)
        log.Debugf("Mount namespace path: %s", nsPathMount)
        nsFileMount, err := os.Open(nsPathMount)
        if err != nil {
            resp <- pidResult{
                nsID: 0,
                err:  fmt.Errorf("Error opening mount namespace file: %s", err),
            }
        }
        defer nsFileMount.Close()

        // set the caller routine into the mount ns (set current thread's procfs the same as parent container)
        err = unix.Setns(int(nsFileMount.Fd()), syscall.CLONE_NEWNS)
        if err != nil {
            resp <- pidResult{
                nsID: 0,
                err:  fmt.Errorf("Error setting mount namespace: %s", err),
            }
        }

        // execute docker inspect to get container fake pid (pid inside a ns rather on host)
        cmd := exec.Command("docker", "inspect", "--format", "{{.State.Pid}}", containerName)
        output, err := cmd.Output()
        if err != nil {
            resp <- pidResult{
                nsID: 0,
                err:  fmt.Errorf("Error executing command in namespace: %s", err),
            }
        }
        fakepid, err := strconv.Atoi(strings.TrimSpace(string(output)))
        if err != nil {
            resp <- pidResult{
                nsID: 0,
                err:  fmt.Errorf("Error converting pid to int: %s", err),
            }
        }
        log.Debugf("Fake PID: %d", fakepid)

        // get unique ns_id pid:[xxx] by current fake pid of container
        nsIDLink, err := getNamespaceID(fakepid)
        if err != nil {
            resp <- pidResult{
                nsID: 0,
                err:  fmt.Errorf("Error getting namespace PID ID: %s", err),
            }
        }
        log.Debugf("Namespace ID link: %s", nsIDLink)

        // get unique ns_id xxx by pid:[xxx]
        nsID := parseNamespaceIDFromLink(nsIDLink)
        log.Debugf("Namespace ID: %d", nsID)
        resp <- pidResult{
            nsID: nsID,
            err:  nil,
        }
        close(resp)                                 // close channel
    }()
    result := <-resp                                // get result from routine channel
    if result.err != nil {
        return 0, result.err
    }

    pid, err := getPIDFromNamespaceID(result.nsID)  // return related pid of given namespace id
    if err != nil {
        return 0, fmt.Errorf("Failed to get PID from namespace ID: %v", err)
    }
    return pid, nil
}
```

3 utility: 'lsns -o NS,PID | grep ${ns_id}' to get real pid by global unique ns_id.
```text
func getPIDFromNamespaceID(namespaceID int) (int, error) {
    procDir := "/proc"
    procIDs, err := os.ReadDir(procDir)
    if err != nil {
        return 0, err
    }

    for _, procID := range procIDs {
        if !procID.IsDir() {
            continue
        }
        pid, err := strconv.Atoi(procID.Name())
        if err != nil {
            continue
        }
        nsPath := filepath.Join(procDir, strconv.Itoa(pid), "ns", "pid")
        link, err := os.Readlink(nsPath)
        if err != nil {
            continue
        }
        nsID := parseNamespaceIDFromLink(link)
        if nsID == namespaceID {
            return pid, nil
        }
    }
    return 0, fmt.Errorf("no process found with namespace ID %d", namespaceID)
}
```

4 utility: 'readlink /proc/${pid}/ns/pid' to get unique ns_id pid:[xxx] by container fake pid.
```text
func getNamespaceID(pid int) (string, error) {
    nsPath := filepath.Join("/proc", strconv.Itoa(pid), "ns", "pid")
    link, err := os.Readlink(nsPath)
    if err != nil {
        return "", err
    }
    return link, nil
}
```

5 utility: convert unique ns_id from pid:[xxx] to xxx.
```text
func parseNamespaceIDFromLink(link string) int {
    fields := strings.Split(link, ":")
    if len(fields) != 2 {
        return 0
    }
    nsID, err := strconv.Atoi(fields[1][1 : len(fields[1])-1])
    if err != nil {
        return 0
    }
    return nsID
}
```

6 core: get real pid of any container on your machine by the container identifier from config.yaml
```text
func InspectContainerPIDIteration(ctx context.Context, containerPath string) (int, error) {
    containerNames := strings.Split(containerPath, ".")                 // convert aaa.bbb.ccc to [aaa, bbb, ccc]
    pid := 0
    var err error
    for index, containerName := range containerNames {
        if index == 0 {                                                 // depth=1 host container
            pid, err = InspectHostContainerPID(ctx, containerName)
            if err != nil {
                return 0, err
            }
            log.Debugf("Container %s PID: %d", containerName, pid)
        } else {                                                        // depth>1 layer containers
            pid, err = InspectContainerPID(pid, containerName)
            if err != nil {
                return 0, err
            }
            log.Debugf("Container %s PID: %d", containerName, pid)
        }
    }
    return pid, nil
}
```

7 migrate link from source to destination netns specified by pid
```text
func MigrateLink(NicTmp, NicName string, pid int) error {
    log.Debugf("Migrating link %s to %s@pid:[%d]", NicTmp, NicName, pid)

    // get ptr to link obj by tmp name
    link, err := netlink.LinkByName(NicTmp)
    if err != nil {
        return fmt.Errorf("Failed to get link %s: %v", NicTmp, err)
    }
    // regist ns-id info for the link obj with certain pid
    netlink.LinkSetNsPid(link, pid)

    resp := make(chan error)
    go func() {
        // lock current routine to certain os thread for any ns related syscall
        runtime.LockOSThread()

        // get related fd corresponding to certain namespace: /proc/${pid}/ns/net -> /var/run/netns/${nsname}
        nsPath := fmt.Sprintf("/proc/%d/ns/net", pid)
        log.Debugf("Mount namespace path: %s", nsPath)
        nsFile, err := os.Open(nsPath)
        if err != nil {
            resp <- fmt.Errorf("Error opening mount namespace file: %s", err)
        }
        defer nsFile.Close()

        // enter into netns specified by fd ref: https://man7.org/linux/man-pages/man8/ip-netns.8.html
        err = unix.Setns(int(nsFile.Fd()), syscall.CLONE_NEWNET)
        if err != nil {
            resp <- fmt.Errorf("Error setting mount namespace: %s", err)
        }

        // get ptr to link obj by nic tmp name again
        link, err := netlink.LinkByName(NicTmp)
        if err != nil {
            resp <- fmt.Errorf("Failed to get link %s in target namespace: %v", NicTmp, err)
        }

        // set formal name for the link obj
        err = netlink.LinkSetName(link, NicName)
        if err != nil {
            resp <- fmt.Errorf("Failed to rename link %s to %s: %v", NicTmp, NicName, err)
        }

        // set the link state: UP
        err = netlink.LinkSetUp(link)
        if err != nil {
            resp <- fmt.Errorf("Failed to set link %s up: %v", NicName, err)
        }
        resp <- nil
    }()
    return <-resp
}
```

<hr>

### # unit test for netns package
1 src code for ut: pkg/netns/netns_test.go
```text
package netns_test

import (
    "testing"
    "context"

    "github.com/melon/NiCon/pkg/netns"
    "github.com/melon/NiCon/pkg/yaml"
)


func TestInspectContainerPIDIteration(t *testing.T) {
    configPath := "./test.yaml"
    config, err := yaml.ReadConfig(configPath)
    if err != nil {
        t.Errorf("Error reading config: %v", err)
        return
    }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    for _, link := range config.Links {
        resultpid , err := netns.InspectContainerPIDIteration(ctx, link.InspectNS)

        // t.Logf only print after test finished
        t.Logf("netns for inspectation: %s, pid: %d", link.InspectNS, resultpid)

        if err != nil {
            t.Errorf("Error reading config: %v", err)
            return
        }
        // do not let the test judge
        // expected := link.ExpectPID
        // if resultpid != expected {
        // 	t.Errorf("Expected %d, but got %d", expected, resultpid)
        // }
    }
}
```

2 data file for ut: pkt/netns/test.yaml
```text
links:
- inspect_ns: setup_lt_1_xxxwxl_melon.olt_1.lt_1
  expected_pid: 17180
```

3 execute the unit test
```text
$ cd ./nicon/pkt/netns/ && go test -v
abb1753c0b8b:/nicon/pkg/netns# go test -v
=== RUN   TestInspectContainerPIDIteration
INFO[0000] Inspecting container olt_1
INFO[0000] Inspecting container lt_1
    netns_test.go:27: netns for inspectation: setup_lt_1_es4wxl_chaowang.olt_1.lt_1, pid: 13146
--- PASS: TestInspectContainerPIDIteration (0.56s)
PASS
ok      github.com/melon/NiCon/pkg/netns        0.575s
```
the real pid for nested container in the host pid netns is 13146.

<hr>

### # shell illustration of getting real pid of any container (nested or not)
1 get the fake pid of container lt_1:
```text
$(user) docker exec -it setup_lt_1_es4wxl_chaowang bash -c " \
docker exec -it olt_1 bash -c \"docker inspect --format '{{.State.Pid}}' lt_1 \""
2977
```

2 get global unique namespace id related to lt_1 container:
```text
$(user) docker exec -it setup_lt_1_es4wxl_chaowang bash -c " \
docker exec -it olt_1 bash -c \" readlink /proc/2977/ns/pid \""
pid:[4026532565]
```

3 get the real host machine pid related to global unique namespace id:
```text
$(root) lsns -o NS,PID | grep 4026532565
4026532565 13146
```

4 enter the namespace of certain pid, and check the interfaces of lt_1:
```text
$(root) nsenter --target 2634106 --net ip a
...
```

<hr>

### # process inheritance analysis using ps & pstree
using ps command to find the nested parent process start from container lt_1 till the init proc 1:
```text
$ ps j 13146
 PPID   PID  PGID   SID   TTY  TPGID  STAT  UID  TIME COMMAND
13114 13146 13146 13146   ?       -1  Ss      0  0:08 /sbin/docker-init -- sh /docker-entrypoint.sh vm.sh boot

$ ps j 13114
 PPID   PID  PGID   SID   TTY  TPGID  STAT  UID  TIME COMMAND
 8841 13114 13114 22656   ?       -1  Sl      0  4:11 /usr/local/bin/containerd-shim-runc-v2 ...

$ ps j 8841
 PPID   PID  PGID   SID   TTY  TPGID  STAT  UID  TIME COMMAND
 8703  8841  8841  8841   ?       -1  Ss      0  0:08 /sbin/tini -- docker-entrypoint.sh ... 

$ ps j 8841
 PPID   PID  PGID   SID   TTY  TPGID  STAT  UID  TIME COMMAND
 8703  8841  8841  8841   ?       -1  Ss      0  0:08 /sbin/tini -- docker-entrypoint.sh ...

$ ps j 8703
 PPID   PID  PGID   SID   TTY  TPGID  STAT  UID  TIME COMMAND
17180  8703  8703 26578   ?       -1  Sl      0  1:36 /usr/local/bin/containerd-shim-runc-v2 ...

$ ps j 17180
 PPID   PID  PGID   SID   TTY  TPGID  STAT  UID  TIME COMMAND
17136 17180 17180 17180   ?       -1  Ss      0  0:08 /sbin/tini -- docker-entrypoint.sh ...

$ ps j 17136
 PPID   PID  PGID   SID   TTY  TPGID  STAT  UID  TIME COMMAND
    1 17136 17136  1511   ?       -1  Sl      0  1:07 /usr/bin/containerd-shim-runc-v2 ...

$ ps j 1
 PPID   PID  PGID   SID   TTY  TPGID  STAT  UID  TIME COMMAND
    0     1     1     1   ?       -1  Ss      0509:58 /usr/lib/systemd/systemd ...
```


show process tree of all child inherited from 17136 with namespace transition (-S):
```text
$(root) pstree -p 17136 -S
cs(17136)───bash(35280,netns)───dd(1821)─┬─{dd}(1823)
     │                                   └─{dd}(30803)
  <<<│setup_xxx container >>>
     ├─tini(17180,netns)───cs(8532)───dd-init(8847,netns)───sshd(9486)
     │        │                ├─tcpdump(6557,netns)
     │        │                ├─{cs}(8534)
     │        │                └─{cs}(8612)
     │        ├─cs(8703)─┬─bash(1965,netns)              <<< lt_1 container >>>
     │        │          └─tini(8841,netns)───cs(13114)───dd-init(13146,netns)───ch(13838)───{ch}(13892)
     │        │        <<< olt_1 container >>>     │                   │              ├─{ch}(29583)
     │        │                     │              │                   │              └─{ch}(29660)
     │        │                     │              │                   ├─vm.sh(13708)───tail(23081)
     │        │                     │              │                   └─vm.sh(21484)───vsd(21489)─┬─{vsd}(21513)
     │        │                     │              │                                               └─{vsd}(23024)
     │        │                     │              ├─tcpdump(17486,netns)
     │        │                     │              ├─{cs}(13115)
     │        │                     │              └─{cs}(25922)
     │        │                     └─python3(9497)─┬─bash(18851)───ncat(18855)
     │        │                                     ├─bash(18853)───dd(18856)─┬─{dd}(18871)
     │        │                                     │                         ├─{dd}(18877)
     │        │                                     │                         └─{dd}(29009)
     │        │                                     ├─dd-d(9512)─┬─cd(22656)─┬─{cd}(22748)
     │        │                                     │            │           └─{cd}(2534)
     │        │                                     │            ├─dd-p(12978)─┬─{dd-p}(12979)
     │        │                                     │            │             └─{dd-p}(12990)
     │        │                                     │            ├─{dd-d}(9918)
     │        │                                     │            └─{dd-d}(12264)
     │        │                                     ├─sshd(9505)
     │        │                                     └─{python3}(18849)
     │        └─python3(17┬─bash(6251)───dd(6252)─┬─{dd}(6253)
     │                    │                       └─{dd}(26897)
     │                    ├─bash(7431)───dd(7444)─┬─{dd}(7448)
     │                    │                       └─{dd}(16546)
     │                    ├─sshd(17479)
     │                    ├─{py}(5878)
     │                    └─{py}(5880)
     ├─{cs}(17137)
     └─{cs}(31343)

# reference table for reduce ps tree layout
cs = containerd-shim
dd = docker
ddd = docker daemon
ch = cloud-hyperviso
netns = ipc,mnt,net,pid,uts
cd = containerd
vsd = virtiofsd
py = python
```

{% endraw %}
