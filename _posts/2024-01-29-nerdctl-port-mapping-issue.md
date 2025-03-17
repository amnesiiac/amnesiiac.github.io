---
layout: post
title: "nerdctl container runtime manager port mapping issue & solution (container)"
author: "melon"
date: 2024-01-29 21:28
categories: "2024"
tags:
  - container
  - network
---

{% raw %}

### # issue: no route to host when ssh to port exposed by nerdctl hostfw container
using nerdctl 1.5.0 as hostfw outmost container cli, after hostfw reaches ready state,
try ssh to the exposed port, but returned err:

```text
$ ssh root@127.0.0.1 -p 49156
ssh: connect to host 127.0.0.1 port 49156: No route to host
```

<hr>

### # issue analysis
1 check the sshd and ssh state, ssh work fine.  
2 by no route to host, first check the layer 2 connectivity by:

```text
$ ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.074 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.059 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.078 ms
64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.053 ms
^C
— 127.0.0.1 ping statistics —
4 packets transmitted, 4 received, 0% packet loss, time 3060ms
rtt min/avg/max/mdev = 0.053/0.066/0.078/0.010 ms
```

thus the localhost is in fine state.  
3 check is certain proc is listening ssh connection on port 49156:

```text
$ netstat -tuln   # show all the listening port list
...
```

from the result, we cannot find the listening port named 49156, which is weird & abnormal.  
4 check machine networking function by nginx server, then ping itself:

```text
$ nerdctl run -d --name melonginx nginx
$ container_ip=$(nerdctl inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' melonginx)
$ curl $container_ip:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

the result shows that local server's layer3 networking function is fine.  
5 by err reported as no route to host, which seems a routing problem, thus need to check the NAT table:

```text
$ iptables -L -n -t nat
```

the rules are configured by nerdctl and kalico cni plugin for k8s pod.
the kalico rules can work well with nerdctl together, but docker cannot work with nerdctl,
which will cause iptables rules conflicts.
howevere, the iptables from the issue env seems fine with no flaws.

<hr>

### # a bit closer to the fact: nerdctl container to host the port mapping has failures
the problematic nerdctl container seems failed to establish the ssh port correctly, meantime,
another nerdctl container which already launched in env has port mappings overlapped with the failed one,
which make current nerdctl hostfw instance failed to launch correctly.
the old & new instance are exposing duplicated ports, so the newer nerdctl container failed to listen on the port.

<hr>

### # expected behavior (tested with nerdctl source code 1.7.2, fix delivered version)
startup nginx container with port 80 and 90 exposed to host:

```text
$ nerdctl run -p :80 -p 90 -d nginx
docker.io/library/nginx:latest:    resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:4c0fdaa8b6341b...:    done           |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:161ef4b1bf7...:    done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:a8758716bb6aa...:    done           |++++++++++++++++++++++++++++++++++++++|
...
elapsed: 56.7s                     total:  67.3 M (1.2 MiB/s)
8c1f125ca497ec95ce617dcceaab3977b73b37bf4a7389b2538bf2b047838cfd

$ nerdctl run -p :80 -p 90 -d nginx
34b5172ec43d03a9a5b3fd965663f188e7e0754047ac355c3c8f5dab2ee6e7fd
```

check the port mapping from nerdctl nginx container to host:

```text
$ nerdctl ps -a
CONTAINER ID  IMAGE         COMMAND         CREATED    STATUS  PORTS                                         NAMES
34b5172ec43d  nginx:latest  "/docker-ent…"  9 sec ago  Up      0.0.0.0:49155->80/tcp, 0.0.0.0:49156->90/tcp  nginx-34b51
8c1f125ca497  nginx:latest  "/docker-ent…"  3 min ago  Up      0.0.0.0:49153->80/tcp, 0.0.0.0:49154->90/tcp  nginx-8c1f1
```

inspect some ip settings of the started containers:

```text
$ nerdctl inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' 34b5172ec43
10.4.0.3

$ nerdctl inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' 8c1f125ca49
10.4.0.2

$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: nerdctl0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 6e:43:e7:d6:2d:10 brd ff:ff:ff:ff:ff:ff
3: veth9c455e0d@nerdctl0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master nerdctl0 state UP
    link/ether fa:65:ab:9b:1c:4a brd ff:ff:ff:ff:ff:ff
4: vethbf3c5f0d@nerdctl0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master nerdctl0 state UP
    link/ether 7a:fc:00:fd:e7:e3 brd ff:ff:ff:ff:ff:ff
1949: eth0@if1950: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:0f brd ff:ff:ff:ff:ff:ff

$ brctl show
bridge name     bridge id               STP enabled     interfaces
nerdctl0        8000.6e43e7d62d10       no              veth9c455e0d
```

we could conclude that the automatically generated host ports are not overlapped, which can be confirmed
using iptables commands as following steps.  
1) the 1st chain is a pre-routing chain, will deal with all pkts and redirect them.

```text
$ iptables -n -v -L -t nat
Chain PREROUTING (policy ACCEPT 1 packets, 67 bytes)
 pkts bytes target             prot opt in  out  source     destination
    2   104 CNI-HOSTPORT-DNAT  0    --  *   *    0.0.0.0/0  0.0.0.0/0    ADDRTYPE match dst-type LOCAL
```

2) then all matched pkts from any itf in to any itf out, from any src ip:port to any dest ip:port (type=LOCAL),
will be pre-route to CNI_HOSTPORT_DNAT chain.

```text
Chain CNI-HOSTPORT-DNAT (2 references)
 pkts bytes target                        prot opt in  out  source     destination
    0     0 CNI-DN-394b84dda006836f40db7  6    --  *   *    0.0.0.0/0  0.0.0.0/0
    /* dnat name: "bridge" id: "default-8c1..." */ multiport dports 49153,49154
    0     0 CNI-DN-ffe9de0bbb4c99ef35271  6    --  *   *    0.0.0.0/0  0.0.0.0/0
    /* dnat name: "bridge" id: "default-34b..." */ multiport dports 49155,49156
```

3) all pkts from from any itf to any itf out, from any src ip:port to any ip with dports in 49153,49154
will be route to CNI_DB_xxxx chain (e.g. 1st container).

```text
Chain CNI-DN-394b84dda006836f40db7 (1 references)
 pkts bytes target                prot opt in   out   source        destination
    0     0 CNI-HOSTPORT-SETMARK  6    --  *    *     10.4.0.0/24   0.0.0.0/0     tcp dpt:49153
    0     0 CNI-HOSTPORT-SETMARK  6    --  *    *     127.0.0.1     0.0.0.0/0     tcp dpt:49153
    0     0 DNAT                  6    --  *    *     0.0.0.0/0     0.0.0.0/0     tcp dpt:49153 to:10.4.0.2:80
    0     0 CNI-HOSTPORT-SETMARK  6    --  *    *     10.4.0.0/24   0.0.0.0/0     tcp dpt:49154
    0     0 CNI-HOSTPORT-SETMARK  6    --  *    *     127.0.0.1     0.0.0.0/0     tcp dpt:49154
    0     0 DNAT                  6    --  *    *     0.0.0.0/0     0.0.0.0/0     tcp dpt:49154 to:10.4.0.2:90

Chain CNI-DN-ffe9de0bbb4c99ef35271 (1 references)
 pkts bytes target                prot opt in   out   source        destination
    0     0 CNI-HOSTPORT-SETMARK  6    --  *    *     10.4.0.0/24   0.0.0.0/0     tcp dpt:49155
    0     0 CNI-HOSTPORT-SETMARK  6    --  *    *     127.0.0.1     0.0.0.0/0     tcp dpt:49155
    0     0 DNAT                  6    --  *    *     0.0.0.0/0     0.0.0.0/0     tcp dpt:49155 to:10.4.0.3:80
    0     0 CNI-HOSTPORT-SETMARK  6    --  *    *     10.4.0.0/24   0.0.0.0/0     tcp dpt:49156
    0     0 CNI-HOSTPORT-SETMARK  6    --  *    *     127.0.0.1     0.0.0.0/0     tcp dpt:49156
    0     0 DNAT                  6    --  *    *     0.0.0.0/0     0.0.0.0/0     tcp dpt:49156 to:10.4.0.3:90
```

3.1) take the #1 rule inside the first chain (CNI-DN-394b84dda006836f40db7) for illustration:  
all pkts from any itf to any itf out, from src subnet 10.4.0.0/24 or 127.0.0.1
to any dest ip with port=49153 will be route to CNI_HOSTPORT_SETMARK chain.

3.2) take the #3 rule inside the first chain (CNI-DN-394b84dda006836f40db7) for illustration:  
all pkts from any itf to any itf out, from src ip:port will be send to DNAT chain for
translation to ${ip_container1}:80. that is a port mapping enablement rule.

<hr>

### # actual behavior (tested with nerdctl 1.5.0)
check the container running state:

```text
$ nerdctl ps -a
CONTAINER ID  IMAGE         COMMAND        CREATED   STATUS  PORTS                                         NAMES
33bc62e01f71  nginx:latest  "/entrypo..."  57s ago   Up      0.0.0.0:49154->80/tcp, 0.0.0.0:49155->90/tcp  nginx-33bc6
c4ae9168d1d5  nginx:latest  "/entrypo..."  4min ago  Up      0.0.0.0:49153->80/tcp, 0.0.0.0:49154->90/tcp  nginx-c4ae9
```

check the port mapping of each container:

```text
$ nerdctl container port c4ae91
80/tcp -> 0.0.0.0:49153
90/tcp -> 0.0.0.0:49154
$ nerdctl container port 33bc62
80/tcp -> 0.0.0.0:49154
90/tcp -> 0.0.0.0:49155
```

try to curl the nginx server, which returned err:

```text
$(alpine) curl 0.0.0.0:49155
<!-- IE friendly error message walkround.
     if error message from server is less than
     512 bytes IE v5+ will use its own error
     message instead of the one returned by
     server.                                 -->

<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01//EN">
<html>
  <head>
  ...
  <head>
<html>
```

check the ip of the container started:

```text
$ nerdctl inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' 33bc62
10.4.0.3
$ nerdctl inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' c4ae91
10.4.0.2
```

check the nat (port mapping) tables of current live container instances:

```text
$ iptables -n -v -L -t nat
Chain PREROUTING (policy ACCEPT 2 packets, 134 bytes)
 pkts bytes target             prot opt in  out     source  destination
    2   104 CNI-HOSTPORT-DNAT     0  --  *    *  0.0.0.0/0    0.0.0.0/0  ADDRTYPE match dst-type LOCAL
...
```

then we turn to chain CNI_HOSTPORT_DNAT for details of port alignment:

```text
Chain CNI-HOSTPORT-DNAT (2 references)
 pkts bytes target        prot opt in  out  source     destination
    0     0 CNI-DN-ce...     6  --  *    *  0.0.0.0/0  0.0.0.0/0
    /* dnat name: "bridge" id: "default-c4ae..." */ multiport dports 49153,49154
    0     0 CNI-DN-83...     6  --  *    *  0.0.0.0/0  0.0.0.0/0
    /* dnat name: "bridge" id: "default-33bc..." */ multiport dports 49154,49155
```

hence, we can see that the dport rule for pkts dest is overlapped, the rules is established
by nerdctl.
it's likely when nerdctl try to add the second port mapping rules, which neglect the multiple
port allocated by the existed container.  
to make it clear, we could try launch 2 container each with only 1 port exposed:

```text
$ nerdctl run -p :80 -d nginx
3b3ca23b527b2f1de6eab8485dd58a7f3c741050c8cc934feb5b640164886517

$ nerdctl run -p :80 -d nginx
6949ed37ae365d75b0af9b5632c8eceaede75c8eedab68bee80bc20ee4c05f43

$ nerdctl ps -a
CONTAINER ID  IMAGE         COMMAND       CREATED         STATUS  PORTS                  NAMES
3b3ca23b527b  nginx:latest  "/entryp..."  50 seconds ago  Up      0.0.0.0:49153->80/tcp  nginx-3b3ca
6949ed37ae36  nginx:latest  "/entryp..."  34 seconds ago  Up      0.0.0.0:49154->80/tcp  nginx-6949e
```

check related iptable rules:

```text
$ iptables -n -v -L -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target             prot opt in  out  source     destination
    2   104 CNI-HOSTPORT-DNAT  0    --  *   *    0.0.0.0/0  0.0.0.0/0   ADDRTYPE match dst-type LOCAL
```
```text
Chain CNI-HOSTPORT-DNAT (2 references)
 pkts bytes target                        prot opt in  out  source     destination
    0     0 CNI-DN-dcd63680852bb51d651b6  6    --  *   *    0.0.0.0/0  0.0.0.0/0
    /* dnat name: "bridge" id: "default-3b3..." */ multiport dports 49153
    0     0 CNI-DN-85677b2b3d40565671dfd  6    --  *   *    0.0.0.0/0  0.0.0.0/0
    /* dnat name: "bridge" id: "default-694..." */ multiport dports 49154
```
everything seems fine with 1 port exposed, so the nerdctl port allocation part seems failed
to notice the NAT rules with multiple dport configured.

<hr>

### # code walk in nerdctl 1.5.0 source code
check the port allocation relatedd dirs:
```text
$(nerdctl/pkg/portutil) tree .
.
├── iptable
│   ├── iptables.go
│   ├── iptables_linux.go
│   └── iptables_test.go
├── port_allocate_linux.go
├── port_allocate_others.go
├── portutil.go
├── portutil_test.go
└── procnet
    ├── procnet.go
    ├── procnet_linux.go
    └── procnetd_test.go

2 directories, 10 files
```
inside alpine container, run original test for the port util pkg:
```text
$(/nerdctl/pkg/portutil) # go test
go: downloading github.com/containerd/go-cni v1.1.9
...
PASS
ok      github.com/containerd/nerdctl/pkg/portutil              0.032s

$(/nerdctl/pkg/portutil/procnet) go test
go: downloading gotest.tools/v3 v3.5.0
go: downloading github.com/google/go-cmp v0.5.9
PASS
ok      github.com/containerd/nerdctl/pkg/portutil/procnet      0.038s

$(/nerdctl/pkg/portutil/iptable) go test
PASS
ok      github.com/containerd/nerdctl/pkg/portutil/iptable      0.010s
```

modify the code to add one more rules to test for multi dports iptables rules parsing:
```text
$(/nerdctl/pkg/portutil/iptable) cat iptables_test.go
// go fmt this script before run
package iptable

import (
    "testing"
)

// test for function: ParseIPTableRules
func TestParseIPTableRules(t *testing.T) {
    // define a struct for test case
    testCases := []struct {
        name  string   // test case name
        rules []string // test rules
        want  []uint64 // wanted test result
    }{
        {
            name:  "Empty input",
            rules: []string{},
            want:  []uint64{},
        },
        {
            name: "Single rule with single port",
            rules: []string{
                "-A CNI-HOSTPORT-DNAT -p tcp -m comment" +
                " --comment \"dnat name: \"bridge\" id: \"some-id\"\" -m multiport --dports 8080 -j CNI-DN-some-hash",
            },
            want: []uint64{8080},
        },
        {
            name: "Multiple rules with multiple ports",
            rules: []string{
                "-A CNI-HOSTPORT-DNAT -p tcp -m comment" +
                " --comment \"dnat name: \"bridge\" id: \"some-id\"\" -m multiport --dports 8080 -j CNI-DN-some-hash",
                "-A CNI-HOSTPORT-DNAT -p tcp -m comment" +
                " --comment \"dnat name: \"bridge\" id: \"some-id\"\" -m multiport --dports 9090 -j CNI-DN-some-hash",
            },
            want: []uint64{8080, 9090},
        },
        {
            name: "Single rule with multiple ports",    // manually add this edging case
            rules: []string{
            "-A CNI-HOSTPORT-DNAT -p tcp -m comment" +
            " --comment \"dnat name: \"bridge\" id: \"some-id\"\" -m multiport --dports 8080,9090 -j CNI-DN-some-hash",
            },
            want: []uint64{8080, 9090},
        },
    }

    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            got := ParseIPTableRules(tc.rules)
            if !equal(got, tc.want) {
                t.Errorf("ParseIPTableRules(%v) = %v; want %v", tc.rules, got, tc.want)
            }
        })
    }
}

// self defined helper function for compare parsed res & wanted res
func equal(a, b []uint64) bool {
    if len(a) != len(b) {
        return false
    }
    for i, v := range a {
        if v != b[i] {
            return false
        }
    }
    return true
}
```
hence, we got the expected err when run the above code test:
```text
$(/nerdctl/pkg/portutil/iptable) go test -v
=== RUN   TestParseIPTableRules
=== RUN   TestParseIPTableRules/Empty_input
=== RUN   TestParseIPTableRules/Single_rule_with_single_port
=== RUN   TestParseIPTableRules/Multiple_rules_with_multiple_ports
=== RUN   TestParseIPTableRules/Single_rule_with_multiple_ports
    iptables_test.go:64: ParseIPTableRules(
    [-A CNI-HOSTPORT-DNAT -p tcp -m comment --comment "dnat name: "bridge" id: "some-id""
    -m multiport --dports 8080,9090 -j CNI-DN-some-hash]) = [8080]; want [8080 9090] --- FAIL: TestParseIPTableRules
    --- PASS: TestParseIPTableRules/Empty_input (0.00s)
    --- PASS: TestParseIPTableRules/Single_rule_with_single_port (0.00s)
    --- PASS: TestParseIPTableRules/Multiple_rules_with_multiple_ports (0.00s)
    --- FAIL: TestParseIPTableRules/Single_rule_with_multiple_ports (0.00s)
FAIL
exit status 1
FAIL    github.com/containerd/nerdctl/pkg/portutil/iptable      0.018s
```
that is: we want to decode the iptable config rules as [8080,9090], but the parsed result is 8080.  

check the iptable port parsing functions:  
the function take a slice of iptable rules as input, return uint64 port list utilized by the rules
```text
package iptable

import (
    "regexp"
    "strconv"
)

func ParseIPTableRules(rules []string) []uint64 {
    ports := []uint64{}
    dportRegex := regexp.MustCompile(`--dports (\d+)`)            // get compiled regex to match the --dports number
    for _, rule := range rules {                                  // for each rule
        matches := dportRegex.FindStringSubmatch(rule)            // return []string of matches, submatches
        if len(matches) > 1 {                                     // if find at least 1 match
            port64, err := strconv.ParseUint(matches[1], 10, 16)  // convert str to uint
            if err != nil {
                continue
            }
            ports = append(ports, port64)                         // add port to list
        }
    }
    return ports
}
```
the above regex only matches for \--dports single_port_num, but cannot parse 1 iptables rule contains multiple dports.

<hr>

### # fix code
replace the iptables.go as our enriched version of code:
```text
package iptable

import (
    // "fmt"
    "regexp"
    "strconv"
    "strings"
)

func ParseIPTableRules(rules []string) []uint64 {
    ports := []uint64{}
    dportRegex := regexp.MustCompile(`--dports ((,?\d+)+)`)          // matches the pattern like: (,)9999,8888...
    // fmt.Printf("%s\n", dportRegex)
    for _, rule := range rules {
        matches := dportRegex.FindStringSubmatch(rule)
        // fmt.Println(matches)
        if len(matches) > 1 {
            for _, _match := range strings.Split(matches[1], ",") {  // iterate over the biggest submatch
                port64, err := strconv.ParseUint(_match, 10, 16)
                if err != nil {
                    continue
                }
                ports = append(ports, port64)                        // add to already allocated port list
            }
        }
    }
    return ports
}
```
which will certainly work for arbitary port exposed for one container under nerdctl management:
```text
=== RUN   TestParseIPTableRules
=== RUN   TestParseIPTableRules/Empty_input
=== RUN   TestParseIPTableRules/Single_rule_with_single_port
=== RUN   TestParseIPTableRules/Multiple_rules_with_multiple_ports
=== RUN   TestParseIPTableRules/Single_rule_with_multiple_ports
--- PASS: TestParseIPTableRules (0.00s)
    --- PASS: TestParseIPTableRules/Empty_input (0.00s)
    --- PASS: TestParseIPTableRules/Single_rule_with_single_port (0.00s)
    --- PASS: TestParseIPTableRules/Multiple_rules_with_multiple_ports (0.00s)
    --- PASS: TestParseIPTableRules/Single_rule_with_multiple_ports (0.00s)
PASS
ok      github.com/containerd/nerdctl/pkg/portutil/iptable      0.010s
```

<hr>

### # reference
containerized nerdctl + containerd + runc: https://github.com/slothfull/containerized-nerdctl

{% endraw %}
