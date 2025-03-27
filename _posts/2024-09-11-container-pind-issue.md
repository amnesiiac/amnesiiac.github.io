---
layout: post
title: "issue when switch from dind to pind (nested container, podman)"
author: "melon"
date: 2024-09-11 21:58
categories: "2024"
tags:
  - container
---

this article focus on introducing the init-system of container runtime.

<hr>

### # background
a hostfw startup issue occur recently after we bump the dind as pind for the setup & olt containers,
in which the podman report the volume it trying to reuse is in wrong state.
the previous volume it detected & try to reuse is created by 'docker in nerdctl', while the error occur
at the env of 'podman in nerdctl'.

the reason why we switch from dind to pind, please refer to devkit pind_vs_dind.png.

<hr>

### # issue description
hostfw the legacy use a dind or d in nertctl container to provide isolation for the emulation part env,
while after we switch to pind container, the issue below block us from startup:

inside build_devices.log of start-ls-host container, in which the setup container is created, the existing
volume is listed in the log:

```text
[20250319 19:11:09 Thread-1 (device_create)     INFO]: device[setup_shelf_boards_1_dkmtj4_root] copy user sshkey finished
[20250319 19:11:09 Thread-1 (device_create)    DEBUG]: run cmd: umask 0 && nerdctl volume ls
[20250319 19:11:09 Thread-1 (device_create)    DEBUG]: run cmd wait_timeout: None
[20250319 19:11:09 Thread-1 (device_create)    DEBUG]: ======================>>
[20250319 19:11:09 Thread-1 (device_create)    DEBUG]: cmd output:
VOLUME NAME      DIRECTORY  <--- an existing vol on work machine, contain reusable docker img to pass into setup container
dind_root_001    /var/lib/nerdctl/1935db59/volumes/default/dind_root_001/_data

...

[20250319 19:11:09 Thread-1 (device_create)     INFO]: run cmd: umask 0 && nerdctl run --name setup_shelf_boards_1_j4_root
--cap-add NET_ADMIN --cap-add NET_RAW
-v /home/fnmst001/devtools/images:/home/fnmst001/devtools/images
-v /home/fnmst001/devtools:/home/fnmst001/devtools:ro
-v /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None:/home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None
-e GET_SETTINGS_IDENTIFIER
-v /etc/issue:/etc/issue
-e WORKDIR=/home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None/setup_shelf_boards_1_dkmtj4_root -e TYPE=SETUP
-v /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None/values.yaml:/host/values.yaml
-e ISAM_SITE_OVERRIDE=Antwerp --ulimit msgqueue=-1
-v /usr/share/zoneinfo/Asia/Kolkata:/usr/share/zoneinfo/Asia/Kolkata:ro
-v dind_root_001:/var/lib/docker
-d -e NICON_URL= -e DOCKER_TLS_CERTDIR=""
--privileged --cgroupns=private
-p :15100/tcp -p :14100/tcp -p :14022/tcp -p :13100/tcp
-p :13022/tcp -p :12022/tcp -p :8222/tcp -p :8024/tcp -p :7222/tcp -p :7024/tcp -p :6222/tcp -p :6024/tcp
-p :3022/tcp -p :2022/tcp -p :1848/tcp -p :1847/tcp -p :1846/tcp -p :1845/tcp -p :4022/tcp -p :5022/tcp
-p :6022/tcp -p :7022/tcp -p :3255/tcp -p :48383/tcp -p :2112/tcp
hostfw-local.artifactory-espoo1.int.net.nokia.com/release/clicktest_pind:v1.3.2
/home/fnmst001/devtools/hostfw/pkg_host/cmds/build_devices.py
--workdir /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None/setup_shelf_boards_1_dkmtj4_root
--setup_model_file /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None/models/setup_shelf_boards_1_dkmtj4_root.yaml
--modeldir /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None/models/setup_shelf_boards_1_dkmtj4_root
--build /home/fnmst001/devtools/images
--no_tcpdump
--no-infra-check
[20250319 19:11:11 Thread-1 (device_create)     INFO]: run cmd wait_timeout: None
[20250319 19:11:11 Thread-1 (device_create)     INFO]: ======================>>
[20250319 19:11:11 Thread-1 (device_create)     INFO]: cmd output:
37105ba43d9bca21cb6e2f00569082b3c015bfc07a3cb611090eb12a766a9727
```

inside build_devices.log of setup container, in which the olt container is created (also a pinp), the error occurred
on the podman run command:

```text
[20250319 13:41:24 Thread-4 (device_create)    DEBUG]: run cmd: umask 0 && podman volume ls
[20250319 13:41:24 Thread-4 (device_create)    DEBUG]: run cmd wait_timeout: None
[20250319 13:41:24 Thread-4 (device_create)    DEBUG]: ======================>>
[20250319 13:41:24 Thread-4 (device_create)    DEBUG]: cmd output:
DRIVER      VOLUME NAME
local       0f479a0093e9a73f3035c28c33a6fabf83afbcbbdb5e6e5ee6191dd5ec115bcb
local       18cf300b1b63698100aaca6f25a7275af930d591d1762a2279b8fbc3b272ef51
local       524ea2fe068d170f4eb45702d669ede7e2343bae588600c8cf3fc9510157b968
local       556d65d5b99ab65c095b8bf7037f5331c8f3cf2c93a406ea50e737d7ba93a2c1
local       774f9b707c8d14446defd2a9c3558c080619e78308429cfde0dd70fdc84a836a
local       a8434e504e28afc44e891640f6c98d27de213d7657e8bd0c53a803c4f34e5ef6
local       ac217e15260d2c2d6024c20d472c865c7d59fae272260d945d298c373d088451
local       c891091efa07ce6f6b9f14aebb6d5b520adcc3f6263280f3fb81d8ff66dbb1f1
local       dind_root_001 <- this volume is passed in by setup container, but it cannot be recognized by podman inside

...

[20250319 13:41:24 Thread-4 (device_create)     INFO]: run cmd: umask 0 && podman run --name olt_1
--cap-add NET_ADMIN --cap-add NET_RAW
-v /home/fnmst001/devtools/images:/home/fnmst001/devtools/images
-v /home/fnmst001/devtools:/home/fnmst001/devtools:ro
-e GET_SETTINGS_IDENTIFIER -v /etc/issue:/etc/issue
-e WORKDIR=/home/fnmst001/host_wo_xgs_None/setup_shelf_boards_1_dkmtj4_root/olt_1
-v /host/values.yaml:/host/values.yaml -e ISAM_SITE_OVERRIueue=-1
-v dind_root_001:/var/lib/docker <----- used to pass outside persistent vol to avoid extra network io for inner docker
-d -e DOCKER_TLS_CERTDIR="" --privileged --cgroupns=private --security-opt seccomp=/etc/containers/hostfw_seccomp.json
-p 15100:12100/tcp -p 14100:11100/tcp -p 14022:11022/tcp -p 13100:10100/tcp -p 13022:10022/tcp -p 12022:9022/tcp
-p 8222:5222/tcp -p 8024:5024/tcp -p 7222:4222/tcp -p 7024:4024/tcp -p 6222:3222/tcp
-p 6024:3024/tcp -p 2022:1022/tcp -p 1848:848/tcp -p 1847:847/tcp -p 1846:846/tcp -p 1845:845/tcp -p 3022:22/tcp
hostfw-local.artifactory-espoo1.int.net.nokia.com/release/clicktest_pind:v1.3.2
/home/fnmst001/devtools/hostfw/pkg_host/cmds/build_devices.py
--workdir /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs/setup_shelf_boards_1_dkmtj4_root/olt_1
--setup_model_file /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None/models/setup_shelf_boards_1_dkmtj4_root/olt_1.yaml
--modeldir /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None/models/setup_shelf_boards_1_dkmtj4_root/olt_1
--build /home/fnmst001/devtools/images
--no_tcpdump
--no-infra-check --smas-mac-bytes ded9578cb573
--rootfs_overlay_post_osw_dir lt_1:/host/mounts/rootfs_overlay_post_osw_ueagx5
[20250319 13:41:25 Thread-4 (device_create)     INFO]: run cmd wait_timeout: None
[20250319 13:41:25 Thread-4 (device_create)     INFO]: ======================>>
[20250319 13:41:25 Thread-4 (device_create)     INFO]: cmd error:  <--- other hostfw instance is taking up the vol resources
Error: container cb1f8855af9ead0576dcad4f7f5a9f7ee560f18f20c08762b9c68086ea6a6801 and volume dind_root_001 share lock ID 0:
deadlock due to lock mismatch
```

setup container logs: create dind (podman in docker) failed:

```text
2025-03-19T13:41:11.664445527Z  * Caching service dependencies ... [ ok ]
2025-03-19T13:41:12.000583949Z ssh-keygen: generating new host keys: ECDSA ED25519 
2025-03-19T13:41:12.211037133Z  * Starting sshd ... [ ok ]
2025-03-19T13:41:12.247962055Z  * Starting Entrypoint ... [ ok ]
2025-03-19T13:41:12.249770469Z net.ipv6.conf.all.disable_ipv6 = 0
2025-03-19T13:41:12.250453059Z fs.inotify.max_user_instances = 512000
2025-03-19T13:41:12.258564028Z xt_CT is installed, only track tftp
2025-03-19T13:41:12.26181232Z start nicon server in setup mode
2025-03-19T13:41:12.279713265Z nicon health response: 
2025-03-19T13:41:17.289605705Z nicon health response: 
2025-03-19T13:41:22.296585371Z nicon health response: {"status":"ok"}
2025-03-19T13:41:22.296669235Z nicon status is OK
2025-03-19T13:41:22.298171365Z clean the stale containers
2025-03-19T13:41:22.379674119Z time="2025-03-19T13:41:22Z" level=error msg="Storage for container
5e0b04935bda5866ee4a82b83e97a9347a555dfaf773515c02009c756a8a67ab has been removed"
2025-03-19T13:41:22.379798916Z time="2025-03-19T13:41:22Z" level=error msg="Storage for container
3b1dd678bc5b342a9dfa46db9f2aa316aaa936b634edc8cf053ef78929e0c8d6 has been removed"
2025-03-19T13:41:22.383476921Z time="2025-03-19T13:41:22Z" level=error msg="Storage for container
ab5aaaad00e7150bd1606632a20353c3b72221f549b75fe157f5a05eced90904 has been removed"
2025-03-19T13:41:22.38534429Z time="2025-03-19T13:41:22Z" level=error msg="Storage for container
d66ad62a7e6396de5a017e8a7c0ec35af4c7a58654df34b2c1db4862a4f6fc1a has been removed"
2025-03-19T13:41:22.386339885Z time="2025-03-19T13:41:22Z" level=error msg="Storage for container
ceee1cf754ad2e9b91702c8259134b19e843f09b1cf162cd238d7dc174d5d393 has been removed"
2025-03-19T13:41:22.393003833Z time="2025-03-19T13:41:22Z" level=error msg="IPAM error:
failed to get network bucket for network bridge"
2025-03-19T13:41:22.406260869Z time="2025-03-19T13:41:22Z" level=error msg="IPAM error:
failed to get network bucket for network bridge"
2025-03-19T13:41:22.406325746Z time="2025-03-19T13:41:22Z" level=error msg="tearing down network namespace
configuration for container d6aa624e7d7130c88e76cd9a8a94c960e25df3b268543a3deb7a059d8b1cce37:
netavark: open container netns: open /run/netns/netns-0de2acda-6a00-ebbf-70ff-0467ad716432: IO error:
No such file or directory (os error 2)"
2025-03-19T13:41:22.406348477Z time="2025-03-19T13:41:22Z" level=error msg="Unable to clean up network for
container d6aa624e7d7130c88e76cd9a8a94c960e25df3b268543a3deb7a059d8b1cce37:
\"unmounting network namespace for container d6aa624e7d7130c88e76cd9a8a94c960e25df3b268543a3deb7a059d8b1cce37:
failed to unmount NS: at /run/netns/netns-0de2acda-6a00-ebbf-70ff-0467ad716432: no such file or directory\""
2025-03-19T13:41:22.410611616Z time="2025-03-19T13:41:22Z" level=error msg="Free container lock:no such file or directory"
2025-03-19T13:41:22.433577186Z time="2025-03-19T13:41:22Z" level=error msg="Free container lock: no such file or directory"
2025-03-19T13:41:22.443405847Z time="2025-03-19T13:41:22Z" level=error msg="Free container lock: no such file or directory"
2025-03-19T13:41:22.44353247Z time="2025-03-19T13:41:22Z" level=error msg="Storage for container
d6aa624e7d7130c88e76cd9a8a94c960e25df3b268543a3deb7a059d8b1cce37 has been removed"
2025-03-19T13:41:22.455819623Z time="2025-03-19T13:41:22Z" level=error msg="Free container lock: no such file or directory"
2025-03-19T13:41:22.469410498Z time="2025-03-19T13:41:22Z" level=error msg="Free container lock: no such file or directory"
2025-03-19T13:41:22.486286007Z time="2025-03-19T13:41:22Z" level=error msg="Free container lock: no such file or directory"
2025-03-19T13:41:22.489769302Z time="2025-03-19T13:41:22Z" level=error msg="Cleaning up volume
(2ee5760dbf8099a724b3f93645c34fe4b1dfe21d004d45e9b7df82021d4a6f37):
freeing lock for volume 2ee5760dbf8099a724b3f93645c34fe4b1dfe21d004d45e9b7df82021d4a6f37: no such file or directory"
2025-03-19T13:41:22.48988326Z d6aa624e7d7130c88e76cd9a8a94c960e25df3b268543a3deb7a059d8b1cce37
2025-03-19T13:41:22.489895581Z ceee1cf754ad2e9b91702c8259134b19e843f09b1cf162cd238d7dc174d5d393
2025-03-19T13:41:22.489900516Z d66ad62a7e6396de5a017e8a7c0ec35af4c7a58654df34b2c1db4862a4f6fc1a
2025-03-19T13:41:22.489904512Z 3b1dd678bc5b342a9dfa46db9f2aa316aaa936b634edc8cf053ef78929e0c8d6
2025-03-19T13:41:22.489942165Z 5e0b04935bda5866ee4a82b83e97a9347a555dfaf773515c02009c756a8a67ab
2025-03-19T13:41:22.489946673Z ab5aaaad00e7150bd1606632a20353c3b72221f549b75fe157f5a05eced90904
2025-03-19T13:41:22.489971421Z Error: container 5534b0d8ddbcbcc90cf81bc5826666ffbdec44102bb1725737bb2fc646b6ab22 has
dependent containers which must be removed before it:
ceee1cf754ad2e9b91702c8259134b19e843f09b1cf162cd238d7dc174d5d393: container already exists
2025-03-19T13:41:22.489997106Z Error: container cfbbb5356fdfa2df53e1e49207e4342025d1d8d2eab8c1296cbc56b0aaec91ee has
dependent containers which must be removed before it:
d66ad62a7e6396de5a017e8a7c0ec35af4c7a58654df34b2c1db4862a4f6fc1a: container already exists
2025-03-19T13:41:22.490002188Z Error: container b6f56771f368162ed75f8fc8dd7b83214175c84ca0ad5e9637082ba1223e5cb4 has
dependent containers which must be removed before it:
5e0b04935bda5866ee4a82b83e97a9347a555dfaf773515c02009c756a8a67ab: container already exists
2025-03-19T13:41:22.490006433Z Error: container 5b9a478618a692c6b4ffd588780f48e144166a53e95d1917cc574bd6fcea61a4 has
dependent containers which must be removed before it:
3b1dd678bc5b342a9dfa46db9f2aa316aaa936b634edc8cf053ef78929e0c8d6: container already exists
2025-03-19T13:41:22.490020754Z Error: container e1f4b4d8bc713fba6f6a2b2255b6acc5e9b68883c72306216fd6e05b215f45b6 has
dependent containers which must be removed before it:
ab5aaaad00e7150bd1606632a20353c3b72221f549b75fe157f5a05eced90904: container already exists
2025-03-19T13:41:22.54959823Z Total reclaimed space: 0B
2025-03-19T13:41:22.562813531Z wait start notify
2025-03-19T13:41:22.562885905Z Executing command: '/home/fnmst001/devtools/hostfw/pkg_host/cmds/build_devices.py
--workdir /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None/setup_shelf_boards_1_dkmtj4_root
--setup_model_file /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None/models/setup_shelf_boards_1_dkmtj4_root.yaml
--modeldir /home/fnmst001/host_workdir/lsfx_fant-g_fwlt-c_xgs_None/models/setup_shelf_boards_1_dkmtj4_root
--build /home/fnmst001/devtools/images
--no_tcpdump
--no-infra-check'
```

<hr>

### # conclusion
after hostfw bump dind to pind, which has impact on existing user's hostfw instance startup.  
1 the obsolete dind created vol is not recognized by current pind setup container.  
2 the previous dind container still consuming the vol, while current container pind has some compatibility break
stuff which lead to the error.

<hr>

### # similar issue reported by podman community
in the failure case env, podman is probably using /tmp/containers-user-$UID (store in system disk storage), and the
dir is not not cleaned up due to some rare reason resulting in container reboot.
then if podman fail to detect a reboot, it will not properly refresh the container state, which will lead to errors
as follows:

```text
Error: container 13fff305a82757d54303f4f627f9749c4e03edda29f91346b417ec90a5e191d7 and volume opendistro share lock ID 0:
deadlock due to lock mismatch
```

ref: https://github.com/containers/podman/issues/9765
