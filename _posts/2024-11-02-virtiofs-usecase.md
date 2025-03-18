---
layout: post
title: "virtiofs usecase: share fs between wrapper container & guest vm (virtualization)"
author: "melon"
date: 2024-11-02 22:14
categories: "2024"
tags:
  - linux
  - virtualization
  - todo
---

virtiofs is a shared file system that lets virtual machine access a directory tree on the host.
unlike existing approaches, it is designed to offer local file system semantics and performance.

<hr>

### # minimal model for virtiofsd usecase (based on hostfw project)
1 modify the cmd_mount function inside vm.sh:

```text
cmd_mount() {
    # start virtiofsd daemon, change the shared dir, change daemon listening sock to clh
    while true; do
        virtiofsd --log-level info --seccomp none --sandbox none --cache=always \
            --socket-path=$WORK_DIR/run/rootextra_test.sock \
            --shared-dir=$WORK_DIR/rootfs_test > $WORK_DIR/run/virtiofsd_test.log 2>&1
        sleep 0.5
        echo "restarting virtiofsd"
    done &
    # pass comm-sock between virtiofsd & cloudhypervisor (for dax mapping add & rm)
    ch-remote --api-socket $WORK_DIR/run/clh.sock add-fs tag=rootextra_test,socket=$WORK_DIR/run/rootextra_test.sock
}
```

2 enter in the wrapper container of vm, execute script, the output shows the virtual dev is created.

```text
xxxx-reborn:/work# mkdir -p rootfs_test
xxxx-reborn:/work# vm.sh mount
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
none on /rootextra/xxxx/user/host/chassis type virtiofs (rw,noatime)
none on /rootextra/mnt/nand-persistent type virtiofs (rw,noatime)
fuse_cpuload on /xxxx/cpuload/config type fuse.fuse_cpuload (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
tmpfs on /xxxx/cpuload/output type tmpfs (rw,relatime,size=1024k)
fuse_devices on /xxxx/slot_default/devs.fuse type fuse.fuse_devices (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
fuse_prozone on /xxxx/slot_default/prozone type fuse.fuse_prozone (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
fuse_devices on /xxxx/slot_default/smas.fuse type fuse.fuse_devices (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
none on /rootextra/mnt/xxxx-nor-mgnt-partition type virtiofs (rw,noatime)
none on /rootextra/mnt/nand-dbase type virtiofs (rw,noatime)
none on /rootextra/xxxx/user/host/shelf type virtiofs (rw,noatime)
mount_cpu, on /mnt/cgroups/cpu type cgroup (rw,relatime,cpu)
mount_cpuacct, on /mnt/cgroups/cpuacct type cgroup (rw,relatime,cpuacct)
fuse_quota on /rootextra/mnt/nand-persistent/persistent/app_data/slot_default.quota type fuse.fuse_quota \
    (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
unionfs on /xxxx/slot_default/run type fuse.unionfs \
    (rw,nosuid,nodev,relatime,user_id=0,group_id=0,default_permissions)
fuse_quota on /rootextra/mnt/nand-persistent/persistent/app_data/nt_1101.quota type fuse.fuse_quota \
    (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
unionfs on /xxxx/slot_1101/run type fuse.unionfs (rw,nosuid,nodev,relatime,user_id=0,group_id=0,default_permissions)
/dev/loop0 on /mnt/xxxx/ZHWTQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop1 on /mnt/xxxx/ZHWYQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop2 on /mnt/xxxx/ZATFQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop3 on /mnt/xxxx/ZJW4QM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop4 on /mnt/xxxx/ZJW5QM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop5 on /mnt/xxxx/ZJXDQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop6 on /mnt/xxxx/ZKK0QM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop7 on /mnt/xxxx/ZKJXQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop8 on /mnt/xxxx/ZKJXQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop9 on /mnt/xxxx/ZK0TQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop10 on /mnt/xxxx/ZK0TQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop11 on /mnt/xxxx/ZK2AQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop12 on /mnt/xxxx/ZK2AQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop13 on /mnt/xxxx/ZK7YQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop14 on /mnt/xxxx/ZK7YQM2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop15 on /mnt/xxxx/ZKSSQD2409.388 type squashfs (ro,relatime,errors=continue)
/dev/loop16 on /mnt/xxxx/ZKSSQM2409.388 type squashfs (ro,relatime,errors=continue)
tmpfs on /mnt/filesync type tmpfs (rw,relatime,size=307200k)
tmpfs on /mnt/dynamic_fast_db type tmpfs (rw,relatime,size=286720k)
sysregd on /mnt/xenomai/anon/root/anon/system type fuse.sysregd \
    (rw,nosuid,nodev,relatime,user_id=0,group_id=0,default_permissions)
tmpfs on /xxxx/logs/internal_syslog type tmpfs (rw,relatime,size=65536k)
resource_monitor on /xxxx/resource_monitor type fuse.resource_monitor (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
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

try mount the fs with the designated tag to the dir created:

```text
$ mkdir /rootextra                                         # create target dir
$ mount rootextra_test /rootextra -t virtiofs -o noatime   # mount tagged fs to target
```

after that the /rootfs_test inside wrapper container is connected with baord vm dir /rootfs_test,
any fs operations will be encapsulated with virtio proto & wrapper pci proto, forwarding to clh,
then pass to virtiofsd by the sock designated, finally the changes will be reflected on both side.

<hr>

### # why need the while true wrapper on virtiofsd (cmd_mount)?
though virtiofsd process is a daemon (acting as socket server), it will exit when vhost-user socket
disconnected with it.

some vmm exceptions (e.g. leading to KVM_EXIT_RESET) might cause the socket disconnection, thus virtiofsd
exit & never come back.
vmm could self resume afterwards, and try to connect with virtiofsd again, while it dont serve anymore,
finally reach connection timeout err.

a real case log illustrating this is as:

```text
20221226T06:47 cloud-hypervisor is ready
20221226T06:47 Host kernel: Linux isam-reborn 5.4.209-1.el7.elrepo.x86_64 #1 SMP Wed Aug3 09:03:41 EDT 2022 x86_64 Linux
20221226T06:47 VM Boot...
20221226T06:47 VM Redirect output to /work/run/boot.log...
20221226T06:48 VMM thread exited with error: Error running command: Server responded with an error:
               Error rebooting VM: InternalServerError: DeviceManagerApiError(ResponseRecv(RecvError))
20221226T06:48 (CreateVirtioFs(VhostUserConnect))
20221226T06:48 Error running command: Error opening HTTP socket: Connection refused (os error 111)
20221226T06:48 VM State: 
20221226T06:48 VM Ready...
20221226T06:48 cloud-hypervisor: 10.301501007s:  WARN:vmm/src/device_manager.rs:1854
               Ignoring error from setting up SIGWINCH listener: Invalid argument (os error 22)
20221226T06:48 cloud-hypervisor: 70.509301809s:  ERROR:virtio-devices/src/vhost_user/vu_common_ctrl.rs:410
               Failed connecting the backend after trying for 1 minute:
               VhostUserProtocol(SocketConnect(Os {code: 111, kind: ConnectionRefused, message: "Connection refused"}))
20221226T06:48 Error running command: Error opening HTTP socket: Connection refused (os error 111)
```

solution to the problem is that: using while true to hold the virtiofsd, when it exit, just resume it.

<hr>

### # reference
virtiofsd intro blog post
