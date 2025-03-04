---
layout: post
title: "cmd to set veth peer out of container (container, veth, ip, netns)"
author: "melon"
date: 2024-08-15 21:08
categories: "2024"
tags:
  - container
---

this article focus on providing a small script to create veth pair inside container,
and use ip command to set one peer of the veth outside the container (to the host machine).

<hr>

### # code in action
inside pkg_host/core/device_builder.py, the following function is used to expose veth peer
outside of container to host machine, which enable the layer 2 network connection between host & container.

```text
def setting_itf_export(self):
    for dev in self.devices.values():
        if dev.dev_type == "SETUP":
            for inf in dev.model.get("network"):
                if inf.get("type") == "inband":
                    # set the veth peer dev outside setup container
                    sh.run("sudo ip netns exec {} ip link set {} down".format(dev.name, inf.get("name")))
                    sh.run("sudo ip netns exec {} ip link set {} netns 1".format(dev.name, inf.get("name")))

                    # assign ip addr to itf dev after set nets (ip/route config will lost when changing netns)
                    for subnet in inf.get("subnets"):
                        ip_cmd = network.get_ip_cmd(subnet.split("/")[0])
                        sh.run("sudo {} addr add {} dev {}".format(ip_cmd, subnet, inf.get("name")))

                    # set the exposed veth peer (in host machine) as up state
                    sh.run("sudo ip link set {} up".format(inf.get("name")))

                    # wait for ipv6 neighbor discovery protocol (NDP) finish
                    time.sleep(5)

                    # setup route according to itf config file
                    for route in inf.get("routes"):
                        ip_cmd = network.get_ip_cmd(route.get("dst").split("/")[0])
                        if route.get("src"):
                            sh.run("sudo {} route add {} src {} dev {}".format(
                                   ip_cmd, route.get("dst"), route.get("src"), inf.get("name"))
                        if route.get("gw"):
                            sh.run("sudo {} route add {} via {} dev {}".format(
                                   ip_cmd, route.get("dst"), route.get("gw"), inf.get("name")))
                    pr.info("moving %s to top layer", inf.get("name"))
```

pinpoint the exact cmd to move veth peer to host machine, safari on the logs:

```text
$(host machine) umask 0 && sudo ip netns exec setup_standalone_board_1_v27n57_root ip link set opr_dbg_li3z netns 1
```

the above cmd is executed on host machine, and it enter in the setup_xxx container, execute ip command to
set the net namespace of veth opr_xxx same as process 1.
does the process 1 mean docker-init or tini inside container? no!
actually this pid 1 denote the one running on the host machine.

the 'ip netns exec' cmd only work with net namespace attached, without involving other namespaces in,
thus the following 'ip link set' is working with host machine's pid/mount namespace.
