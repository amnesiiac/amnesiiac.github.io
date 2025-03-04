---
layout: post
title: "socat tools to enable rpc through vmm & vm (socat, rpc, shared fs)"
author: "melon"
date: 2024-08-17 21:49
categories: "2024"
tags:
  - rpc
  - virtualization
---

this article focus on introduce how to use socat tool for rpc purpose from the wrapper container env
inside the vm managed by the vmm like cloud-hypervisor.

<hr>

### # socat for rpc cross host & guest kernel
1 vm wrapper container setup file & scripts:

```text
./vmm
├── docker-entrypoint-dev.sh
├── docker-entrypoint.sh
├── Dockerfile
├── Dockerfile.dev
├── hot_patch
│   └── init
├── INFO.txt
├── Makefile
├── vm.sh
└── vsexec
```

2 Makefile: download socat tool & store in current workdir:

```text
include INFO.txt

build: hot_patch/usr/bin/socat
        docker build -f Dockerfile -t $(IMAGE_NAME):$(VMM_VERSION) \
			--build-arg http_proxy=$(HTTP_PROXY) --build-arg https_proxy=$(HTTPS_PROXY) .
        docker build -f Dockerfile.dev -t $(IMAGE_NAME):$(VMM_VERSION)-dev \
			--build-arg http_proxy=$(HTTP_PROXY) --build-arg https_proxy=$(HTTPS_PROXY) \
			--build-arg IMAGE_NAME=$(IMAGE_NAME) --build-arg VERSION=$(VMM_VERSION) .

push: build
        docker push $(IMAGE_NAME):$(VMM_VERSION)
        docker push $(IMAGE_NAME):$(VMM_VERSION)-dev

hot_patch/usr/bin/socat:
        mkdir -p ./hot_patch/usr/bin
        wget -Y off -O $@ https://artifactory-espoo1.int.net.xxxxx.com/artifactory/local/apps/socat-1.7.4.1-x86_64
        chmod +x $@
```

3 vsexec: an expect script to spawn socat client inside wrapper container to connect to socat server
in board vm via unix vsock, after connection established, send the cmd to be executed inside vm.

```text
#!/usr/bin/expect -f
set mycommand [lindex $argv 0 ]

spawn socat - UNIX-CONNECT:/tmp/vmm.vsock
    send "CONNECT 1234\r"
    send "${mycommand}\r"
    send "exit\r"
    expect eof
```

4 vm.sh snippet: config network settings for wrapper container & vmm.

```text
cmd_mount() {
    # restart virtiofsd in case it silently exit
    while true; do
        # start virtiofsd with sock for communication, create shared fs between wrapper container & board vm
        virtiofsd --log-level info --seccomp none --sandbox none --cache=always \
--socket-path=$WORK_DIR/run/rootextra.sock --shared-dir=$WORK_DIR/rootfs > $WORK_DIR/run/virtiofsd.log 2>&1
        sleep 0.5
        echo "restarting virtiofsd"
    done &
    # add a virtual device to the vm under control, expose tag inside to be used for mount
    ch-remote --api-socket $WORK_DIR/run/clh.sock add-fs tag=rootextra,socket=$WORK_DIR/run/rootextra.sock
}

cmd_network() {
    local link_local_prefix="fe80"
    ip -br a | sed "s/${link_local_prefix}[^ ]* //g" | while read -r ifname stat addr; do
        if [[ $ifname != "lo" ]]; then
            setup_vm_link ${ifname%@*} $addr
        fi
    done
    # add api-socket & vsock (with file backup) for communication between vmm & vm
    ch-remote --api-socket $WORK_DIR/run/clh.sock add-vsock cid=3,socket=/tmp/vmm.vsock
}
```

5 docker-entrypoint.sh: startup script for wrapper container.

```text
#!/bin/bash

vm.sh start
vm.sh mkrootfs
vm.sh create
vm.sh mount
vm.sh network

exec "$@"
```

the virtiofsd log is as (info level):

```text
[2024-07-24T02:40:01Z INFO  virtiofsd] Waiting for vhost-user socket connection...
[2024-07-24T02:40:46Z INFO  virtiofsd] Client connected, servicing requests
```

as we see the virtiofsd is waiting for vhost-user, which means the fs client is inside userspace
rather than kernel space vhost-net.

6 the early s6 scripts for simulation framework board startup configurations:
/buildroot/board/xxxxx/xxxx/features/xxxx/target_skeleton_extras/etc/init.d/S00early_configure_qemu,
in which basic network settings, socat server for rpc from framework is enabled.

```text
#!/bin/sh

hostfw_vm_init() {
    umask 0

    # done the job as s6-mount of target, neglect the stub part
    is_mounted /dev || mount none /dev -t devtmpfs
    mkdir -p /dev/pts
    is_mounted /dev/pts || mount none /dev/pts -t devpts
    is_mounted /proc || mount none /proc -t proc
    is_mounted /sys || mount none /sys -t sysfs

    # mount the passed in virtual device into fs, shared between container & board
    mkdir /rootextra                                           # create fs
    mount rootextra /rootextra -t virtiofs -o noatime          # mount the dev by tag assigned by virtiofsd to target
    ln -sf /rootextra/mnt/* /mnt                               # overwrite dir from share folderto real
    ln -sf /rootextra/isam/user/host/* /isam/user/host
    ln -sf /rootextra/etc/* /etc
    for f in /rootextra/dev/*; do ln -sf $f ${f#/rootextra}; done

    # use hvc0 as console
    #ln -sf /dev/hvc0 /dev/ttyS0
    #ln -sf /dev/hvc0 /dev/ttyS1
    ln -sf /dev/ttyS0 /dev/ttyS1                               # use ttyS0 as console

    depmod $(uname -r)                                         # regenerate the module dependency info
    setup_network                                              # set sim board network by vmnet.conf file in shared dir
    sed -i /'${1} -f'/i\\'touch /rootextra/.exit \nsleep 5' /etc/rc.shutdown  # support shutdown in cloudhypervisor

    # setup socat server at bg, listen to connection from port 1234:
    # each connection fork new proc, alloc pty, redirect stderr to pty, create new session,
    # forward ctrl+c irq to bash session, disable input echo.
    socat VSOCK-LISTEN:1234,reuseaddr,fork EXEC:bash,pty,stderr,setsid,sigint,sane,ctty,echo=0 &
}

docker_vm_early_init() {
    if [ ! -d "/rootextra/mnt" ]; then                         # if not hostfw not finish init work, do it here
        echo "mount /rootextra"
        hostfw_vm_init                                         # *
    else
        echo "Initialization has been completed by hostfw-vm"  # already init by hostfw
    fi
    rm -f /.dockerenv                                          # unset docker env flag

    for file in /etc/init.d/docker.*; do                       # copy from docker init, unsure whether really needed
        [ -e "$file" ] || break                                # handle the case of no files matching pattern
        mv "$file" $(echo "$file" | sed s/docker.//)
    done
    ip link show eth0 && ifconfig eth0 down                    # set up virtual obc network card
    ip link add eth-obc0 type veth peer name eth-obc1
    ip link set eth-obc0 up && ip link set eth-obc1 up
    sysctl -w fs.inotify.max_user_instances=8192               # configre the inotify open file number
}

prepare_host_env() {
    if [ -d "/rootextra" ]; then
        touch /.vmenv
    fi
    if is_vm_env; then
        docker_vm_early_init                                   # *
    elif is_docker_env; then
        docker_early_init
    else
        qemu_early_init
    fi
}

case "$1" in
    start)
        prepare_host_env                                       # *
        populate_rip_data
        populate_temp_data
        create_persistent_symlinks
        overlay_board_data

        if config_flag_enabled enable_redundancy && [ ! -f /isam/user/host/chassis/eps/eps_self_active ]; then
            mkdir -p /isam/user/host/chassis/eps/
            echo 0 > /isam/user/host/chassis/eps/eps_peer_present
            echo 1 > /isam/user/host/chassis/eps/eps_self_avialiable
            echo 1 > /isam/user/host/chassis/eps/eps_self_active
        fi
        ls /isam/user/host/chassis/eps/
        exit 0
        ;;
    stop) ;;
    *)
        echo "Usage: $0 {start|stop}" >&2
        exit 1
        ;;
esac
```

<hr>

### # usecase 0: try create another shared dir between wrapper container & board image vm
1 modify the cmd_mount function inside vm.sh:

```text
cmd_mount() {
    # start virtiofsd daemon, change the shared dir, change the daemon listening sock
    while true; do
        virtiofsd --log-level info --seccomp none --sandbox none --cache=always \
--socket-path=$WORK_DIR/run/rootextra_test.sock --shared-dir=$WORK_DIR/rootfs_test > \
$WORK_DIR/run/virtiofsd_test.log 2>&1
        sleep 0.5
        echo "restarting virtiofsd"
    done &
    # change the comm between virtiofsd with vmm cloudhypervisor
    ch-remote --api-socket $WORK_DIR/run/clh.sock add-fs tag=rootextra_test,socket=$WORK_DIR/run/rootextra_test.sock
}
```

2 enter in the wrapper container of vm, execute script, the output shows the virtual dev is created.

```text
isam-reborn:/work# mkdir -p rootfs_test
isam-reborn:/work# vm.sh mount
{"id":"_fs8","bdf":"0000:00:0a.0"}
```

3 enter into the board vm, check the existence of the device by navigating the device mounted:

```text
$(board) mount
rootfs on / type rootfs (rw,size=447424k,nr_inodes=1271494)
none on /dev type devtmpfs (rw,relatime,size=4096k,nr_inodes=1271499,mode=755)
none on /dev/pts type devpts (rw,relatime,mode=600,ptmxmode=000)
none on /proc type proc (rw,relatime)
none on /sys type sysfs (rw,relatime)
rootextra on /rootextra type virtiofs (rw,noatime)   // pre established shared dir mounted at /rootextra
none on /rootextra/isam/user/host/chassis type virtiofs (rw,noatime)
none on /rootextra/mnt/nand-persistent type virtiofs (rw,noatime)
fuse_cpuload on /isam/cpuload/config type fuse.fuse_cpuload (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
tmpfs on /isam/cpuload/output type tmpfs (rw,relatime,size=1024k)
fuse_devices on /isam/slot_default/devs.fuse type fuse.fuse_devices (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
fuse_prozone on /isam/slot_default/prozone type fuse.fuse_prozone (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
fuse_devices on /isam/slot_default/smas.fuse type fuse.fuse_devices (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
none on /rootextra/mnt/isam-nor-mgnt-partition type virtiofs (rw,noatime)
none on /rootextra/mnt/nand-dbase type virtiofs (rw,noatime)
none on /rootextra/isam/user/host/shelf type virtiofs (rw,noatime)
mount_cpu, on /mnt/cgroups/cpu type cgroup (rw,relatime,cpu)
mount_cpuacct, on /mnt/cgroups/cpuacct type cgroup (rw,relatime,cpuacct)
fuse_quota on /rootextra/mnt/nand-persistent/persistent/app_data/slot_default.quota type fuse.fuse_quota \
    (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
unionfs on /isam/slot_default/run type fuse.unionfs \
    (rw,nosuid,nodev,relatime,user_id=0,group_id=0,default_permissions)
fuse_quota on /rootextra/mnt/nand-persistent/persistent/app_data/nt_1101.quota type fuse.fuse_quota \
    (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
unionfs on /isam/slot_1101/run type fuse.unionfs (rw,nosuid,nodev,relatime,user_id=0,group_id=0,default_permissions)
/dev/loop0 on /mnt/isam/ZHWTQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop1 on /mnt/isam/ZHWYQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop2 on /mnt/isam/ZATFQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop3 on /mnt/isam/ZJW4QM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop4 on /mnt/isam/ZJW5QM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop5 on /mnt/isam/ZJXDQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop6 on /mnt/isam/ZKK0QM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop7 on /mnt/isam/ZKJXQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop8 on /mnt/isam/ZKJXQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop9 on /mnt/isam/ZK0TQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop10 on /mnt/isam/ZK0TQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop11 on /mnt/isam/ZK2AQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop12 on /mnt/isam/ZK2AQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop13 on /mnt/isam/ZK7YQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop14 on /mnt/isam/ZK7YQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop15 on /mnt/isam/ZKSSQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop16 on /mnt/isam/ZKSSQM2409.388 type squashfs (ro,relatime,errors=continue)
tmpfs on /mnt/filesync type tmpfs (rw,relatime,size=307200k)
tmpfs on /mnt/dynamic_fast_db type tmpfs (rw,relatime,size=286720k)
sysregd on /mnt/xenomai/anon/root/anon/system type fuse.sysregd \
    (rw,nosuid,nodev,relatime,user_id=0,group_id=0,default_permissions)
tmpfs on /isam/logs/internal_syslog type tmpfs (rw,relatime,size=65536k)
resource_monitor on /isam/resource_monitor type fuse.resource_monitor (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
none on /rootextra/ap/local/devtools type virtiofs (rw,noatime)
none on /rootextra/repo1/guolinp/hostcontroller/instances/admin-lant-a_\
    20240715112449/workdir/setup_shelf_boards_1/olt_1/nt_1 type virtiofs (rw,noatime)
```

however, the above output shows: there's no newly created shared fs.
check the device existence by sysfs:

```text
$(board) cd /sys/devices/pci0000:00/0000:00:09.0 && ls
ari_enabled                 dma_mask_bits       local_cpulist     remove              subsystem_vendor
broken_parity_status        driver              local_cpus        rescan              uevent
class                       driver_override     modalias          resource            vendor
config                      enable              msi_bus           resource0           virtio8
consistent_dma_mask_bits    firmware_node       msi_irqs          revision
d3cold_allowed              irq                 power             subsystem
device                      link                power_state       subsystem_device
```

which show the virtual device is added to vm guest kernel.

try mount the fs with the designated tag provided in clh in vm_mount:

```text
$ mount rootextra_test /rootfs_test
```

after that the /rootfs_test inside wrapper container is connected with baord vm dir /rootfs_test,
any fs operations will be encapsulated with virtio proto & wrapper pci proto, forwarding to clh,
then pass to virtiofsd by the sock designated, finally the changes will be reflected on both side.

<hr>

### # usecase 1: framework level syslog collector atexit of vm board
inside device simulation framework, use the vsexec script to help container framework collect board syslogs
before normal exit, intentionally exit, predictable exit, hardware watchdog triggered exit\...

```text
def collect_system_log(self):
    pr.info("dev: {}, collect system logs start".format(self.name))
    # check if reboot log already recorded
    def is_reboot_info_exist():
        status, _ = self.sb.exec(
            'ls /work/rootfs/mnt/nand/reboot_info.dir/reboot_log_*_*_* >/dev/null 2>&1'
        )
        return status == 0

    # for container + vmm + vm + device env
    if self.with_vmm:
        if not is_reboot_info_exist():
            # if the vsexec execute ok (return 0), which means the kernel/socat server working fine (not hang/panic)
            # if uname contains linux, then it's linux board rather than vxworks or other os board.
            # if all condition met, sleep 5 to wait the board continue exec its reboot info collecting process.
            self.sb.exec(
                'if ! vsexec uname | grep -q "Linux"; then '
                '  sleep 5; '
                'fi '
            )
        # collect syslog, reboot info and put them into board persistent dir /rootextra
        self.sb.exec(
            'vsexec "/isam/scripts/collect_logs now; \            # collect logs
             cp -afH /mnt/reboot_info /tmp/system_logs_now; \     # copy reboot info into system log dir
             cp -afH /tmp/system_logs_now /rootextra"'            # copy reboot info & system logs into shared mount
        )
        if self.sb.cp("/work/rootfs/system_logs_now", self.workdir, reverse=True) != 0:
            sh.run("mkdir -p {}".format(self.workdir))
        if is_reboot_info_exist():
            self.sb.cp("/work/rootfs/mnt/nand/reboot_info.dir", self.workdir + "/system_logs_now", reverse=True)
            sh.run("cd {}/system_logs_now && cp -r reboot_info.dir/* reboot_info && rm -rf reboot_info.dir".format(
                   self.workdir))
        sh.run("cd {} && mv system_logs_now system_logs && zip -rq system_logs.zip system_logs".format(self.workdir))
   else :
        self.sb.exec("/isam/scripts/collect_logs now; cp -afH /mnt/reboot_info /tmp/system_logs_now")
        self.sb.cp("/tmp/system_logs_now", self.workdir, reverse=True)
        sh.run("cd {} && mv system_logs_now system_logs && zip -rq system_logs.zip system_logs".format(self.workdir))
    fs.remove_files(os.path.join(self.workdir, "system_logs"))
    pr.info("dev: {}, collect system logs done".format(self.name))
```

<hr>

### # usercase 2: show board vm dir info based on the shared fs
in board wrapper container, the vmm.vsock is established fine:

```text
isam-reborn:/tmp# ls -l
total 0
srwx------    1 root     root             0 Jul 15 05:54 vmm.vsock
```

in board wrapper container, execute ls -l command using vsexec script:

```text
isam-reborn:/# which vsexec
/usr/bin/vsexec

isam-reborn:/# vsexec "ls -l"
spawn socat - UNIX-CONNECT:/tmp/vmm.vsock
CONNECT 1234
ls -l
exit
OK 1073741824
~ # total 16
drwxr-xr-x    2 root     root     1540 Jul 15 05:55 bin
----------    1 root     root        0 Jul 15 05:55 crond.reboot
drwxr-xr-x    7 root     root     3060 Jul 15 05:55 dev
-rw-r--r--    1 root     root       17 Jul 15 05:53 docker.stub
drwxr-xr-x   18 root     root     1380 Jul 15 05:58 etc
drwxr-xr-x    3 fdh      fdh        60 Jul 15 05:53 home
-rwxr-xr-x    1 root     root     2232 Jul 15 05:54 init
drwxrwxrwx   29 root     root      680 Jul 15 05:55 isam
drwxr-xr-x    5 root     root      640 Jul 15 05:55 lib
lrwxrwxrwx    1 root     root        3 Jul 15 05:54 lib32 -> lib
lrwxrwxrwx    1 root     root       11 Jul 15 05:54 linuxrc -> bin/busybox
drwxr-xr-x    2 root     root       40 Jul 15 05:54 log
drwxr-xr-x    2 root     root       40 Jul 15 05:54 media
drwxr-xr-x   10 root     root      380 Jul 15 05:55 mnt
drwxr-xr-x    2 root     root       40 Jul 15 05:54 opt
dr-xr-xr-x  320 root     root        0 Jul 15 05:54 proc
drwx------    3 root     root       80 Jul 17 09:11 root
drwxr-xr-x    1 root     root     4096 Jul 15 05:54 rootextra
drwxr-xr-x   13 root     root     1460 Jul 16 06:00 run
drwxr-xr-x    2 root     root     1280 Jul 15 05:54 sbin
dr-xr-xr-x   12 root     root        0 Jul 15 05:54 sys
drwxrwxrwt   12 root     root      980 Jul 17 09:14 tmp
-r-xr-xr-x    1 root     root      351 Jul 15 05:54 type_a.version
drwxr-xr-x    8 root     root      180 Jul 15 05:54 usr
drwxr-xr-x    7 root     root      260 Jul 15 05:55 var
lrwxrwxrwx    1 root     root       54 Jul 15 05:55 yang_categories_per_mode.json -> /mnt/xxxx.json
```

the board fs dir info is shown by this rpc call, which can be confirmed inside board vm.

<hr>

### # further readings
ref: https://virtio-fs.gitlab.io/index.html  
ref: some blog post with base illustration on virtio
