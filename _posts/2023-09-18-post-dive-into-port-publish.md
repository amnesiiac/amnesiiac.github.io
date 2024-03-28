---
layout: post
title: "dive into container port publishing (container)"
author: "melon"
date: 2023-09-18 20:20
categories: "2023"
tags:
  - container
  - ongoing
---

### # introduction
port publishing is a container concept, while a more traditional network technique is called port forwarding.

<hr>

### # port forwarding (aka port mapping) vs socket direction?
the "port" mentioned inline is indeed a socket address defined by "ip:port", socket is the basic unit for network data exchanging.  
thus, the port forwarding is fancy name for describing the data addressed to one socket is redirected to another another socekt by imtermediary (network router or proxy process).

And technically, port forwarding is a form of NAT(network address translation).

1 socket direction:
```txt
                         ┌────────┐                                       ┌────────┐
                         │ client ├────────>$$$$$$$$$$$$$$$$$$$$$$───────>│ server │
                         └────────┘                                       └────────┘
                   send data to ip:port                                listen on ip:port
```
2 port forwarding:
```txt
                                       direct localhost:port to ip:port
                         ┌────────┐             ┌───────────┐             ┌────────┐
                         │ client ├────────────>│redirection├────────────>│ server │
                         └────────┘             └───────────┘             └────────┘
                   send data to localhost:port                         listen on ip:port
```

<hr>

### # two ways to direct network packet
1) sneakly modify the dest addr of packets.  
the packets originally destined to ip:port is modifed and destined to ip2:port2 (e.g. a netfilter configured with a bunch of iptable rules).  
```txt
              kernel space forwarding
              (iptables/eBPF/LVS...)   kernel rewrite packets' dest addr 
                    ┌────────┐         from localhost:port with ip:port         ┌────────┐
                    │ client ├─────────────$$$$$$$$$$$$$$$$$$$$$$$─────────────>│ server │
                    └────────┘                                                  └────────┘
           send data to localhost:port                                      listen on ip:port
```

2) explicitly putting a proxy between client & server.  
the client-end socket is maintained by >=layer4 proxy process, which read the data and redirect to final destination.  
```txt
                   user space forwarding:    listen on localhost:port
                                             => connect to ip:port
                         ┌────────┐                 ┌───────┐                  ┌────────┐
                         │ client ├─────$$$$$$─────>│ proxy ├─────$$$$$$──────>│ server │
                         └────────┘                 └───────┘                  └────────┘
                   send data to localhost:port                              listen on ip:port
```

<hr>

### # container and port forwarding
By port forwarding technique, we could access the containerized service (e.g. for ad-hoc tasks) using the host machine ip rather than the container ip. Why we need port publishing inside container? There are at least 2 inconvenience of accessing containers' service by their ip:  
1) container ip are assigned dynamically, the restart or recreation of a container might lead to ip changes.  
2) by default, container ip are only routable inside host and unaccessible outside, thus port forwarding provide a way for easy access.

Analysis the container principle of port puiblishing:  
```text
$ docker run -d -p 8080:80 --name nginx-1 nginx
```
container port publishing:
```txt
                            ┌──────────────────────────────────────────────────────┐
                            │ docker host machine                    172.17.0.3    │
                            │ 192.168.1.5                           ┌───────────┐  │
                            │                                    ┌─>│ container │  │
                            │                                    │  └───────────┘  │
                            │            172.17.0.1              │                 │
                            │        ┌────────────────┐          │   172.17.0.4    │
                            │        │    docker0     ├──────────┘  ┌───────────┐  │
                            │        │                ├────────────>│ container │  │
                            │        │ virtual bridge ├──────────┐  └───────────┘  │
                            │        └────────────────┘          │                 │
                            │    ┌──────────────────────────┐    │   172.17.0.5    │
              curl:         │    │        iptables          │    │  ┌───────────┐  │
              host-ip:8080──+────┤ rewrite 192.168.1.5:8080 ├────┴─>│   nginx   │  │
                            │    │      to 172.17.0.5:80    │       └───────────┘  │
                            │    └──────────────────────────┘       listen on 0:80 │
                            └──────────────────────────────────────────────────────┘
```
check NAT table to examine port forwarding rules:
```text
$ sudo iptables -t nat -L
...
Chain POSTROUTING (policy ACCEPT)
target        prot opt source       destination
...
MASQUERADE    tcp  --  172.17.0.2   172.17.0.2     tcp dpt:http

Chain DOCKER (2 references)
target        prot opt source       destination
...
DNAT          tcp  --  anywhere     anywhere       tcp dpt:8080 to:172.17.0.2:80
```

<hr>

### # port publishing using different network driver
todo
ref: https://docs.docker.com/network/drivers/bridge/

<hr>

### # macos docker desktop port publishing
{% raw %}
```text
$ docker run -d --name nginx-test nginx
$ conf_ip=$(docker inspect -f '{{range.NetworkSettings.Network}}{{.IPAddress}}{{end}}') nginx-test
$ ping $conf_ip
PING 172.17.0.3 (172.17.0.3): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
^C
--- 172.17.0.3 ping statistics ---
3 packets transmitted, 0 packets received, 100.0% packet loss
``` 
{% endraw %}
container port publishing on macos:
```txt
                 ┌───────────────────────────────────────────────────────────────────────────┐
                 │ docker host machine (macos)                                               │
                 │                  ┌──────────────────────────────────────────────────────┐ │
                 │                  │ hyperkit VM                            172.17.0.3    │ │
                 │ forwarding to VM │ 192.168.65.1                          ┌───────────┐  │ │
                 │ ┌──────────────┐ │                                    ┌─>│ container │  │ │
                 │ │ docker proxy │ │                                    │  └───────────┘  │ │
                 │ └────+─────┬───┘ │            172.17.0.1              │                 │ │
                 │  listen on 0:80  │        ┌────────────────┐          │   172.17.0.4    │ │
                 │      │     │     │        │    docker0     ├──────────┘  ┌───────────┐  │ │
                 │      │     │     │        │                ├────────────>│ container │  │ │
                 │      │     │     │        │ virtual bridge ├──────────┐  └───────────┘  │ │
                 │      │     │     │        └────────────────┘          │                 │ │
                 │      │     │   VPNKit ┌───────────────────────────┐   │   172.17.0.5    │ │
                 │      │     │   ┌───┐  │        iptables           │   │  ┌───────────┐  │ │
                 │      │     └─────+───>│ rewrite 192.168.65.1:8080 ├───┴─>│   nginx   │  │ │
                 │     curl       └───┘  │      to 172.17.0.5:80     │      └───────────┘  │ │
                 │ localhost:8080   │    └───────────────────────────┘      listen on 0:80 │ │
                 │                  └──────────────────────────────────────────────────────┘ │
                 └───────────────────────────────────────────────────────────────────────────┘
```
docker desktop lower layer implementation principle for port mapping is a combination of user space port forwarding and kernel iptable-based port forwarding.
```txt
                                                    HOST │ LINUX VM
                                                         │
                             bind macOS privileged ports │ 
                                ┌────────────────────┐   │         
                                │ privileged service │   │   forwards to container IP
                                └──────────+─────────┘   │ via veth devices and bridges
┌──────────────────────────┐    ┌──────────┴─────────┐   │    ┌──────────────────┐    ┌───────┐
│curl http://localhost:port├───>│ com.docker.backend ├───┼───>│ vpnkit forwarder ├───>│ nginx │
└──────────────────────────┘    └──────────+─────────┘   │    └──────────────────┘    └───────┘
       ┌─────────────────────┐    ┌────────┴─────────┐   │                            ┌─────────┐
       │ docker run -p 80:80 ├───>│ docker API proxy ├───┼───────────────────────────>│ dockerd │
       └─────────────────────┘    └──────────────────┘   │                            └─────────┘  user space
 ────────────────────────────────────────────────────────┼───────────────────────────────────────────────────
                                     ┌───────────────┐   │    ┌───────────────┐                  kernel space
              forwards unix domain   │ vpnkit bridge ├───┼───>│ vpnkit bridge │
              sockets over AF_VSOCK  └───────────────┘   │    └───────────────┘
```
we could examine the docker user space process on HOST machine by:
```text
# start a nginx server
$ docker run -d -p 8080:80 --name nginx-1 nginx

# pinpoint the process that listen to port 8080
$ sudo lsof -i -P | grep LISTEN | grep :8080
com.docke... 24294   iximiuz   76u  IPv6 0x89404e558d90602b    0t0   TCP *:8080 (LISTEN)

# show the docker API proxy process info
$ ps 24294
  PID   TT  STAT      TIME COMMAND
24294   ??  S     38:09.83 /Applications/Docker.app/Contents/MacOS/com.docker.backend -watchdog -native-api
```
then the details of data transfering using docker desktop is as:  
1) the "com.docker.backend" process on host system acting as a user-space proxy, which create a socket using specified ip:port.  
2) the created socket can accessed by the host network, which is used to send data to container.  
3) data send to the socket, it gets forwarded by vpkkit-bridge to VM's external network interface.  
4) inside the VM, the data is passed by "port forwarding" technique described above.

ref: https://iximiuz.com/en/posts/docker-publish-container-ports/
