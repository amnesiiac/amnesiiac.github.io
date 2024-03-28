---
layout: post
title: "introduction to dts file & uboot fdt tool (linux, buildroot)"
author: "melon"
date: 2024-01-02 20:37
categories: "2024"
tags:
  - linux
  - buildroot 
---

### # what is dts file in in buildroot for uboot
a dts (device tree source) file is used to describe the hardware configuration of a specific device or board.
buildroot is a popular build system used for creating embedded linux systems,
and the dts file plays a crucial role in configuring the device-specific aspects of the linux kernel.

here's how the dts file is used in the buildroot repository:  
1 board configuration:
the dts file contains information about the hardware components and their configuration on a specific board or device.
this includes details about the CPU, memory, peripherals, buses, and other hardware-specific settings.

2 device tree compilation:
buildroot uses the dts file to generate the corresponding device tree blob (dtb) file.
the dtb file is a binary representation of the dts file, which can be understood and processed by the linux kernel.

3 kernel configuration:
the generated dtb file is then used during the linux kernel configuration process.
it is typically included in the kernel image or passed to the bootloader,
enabling the kernel to identify and configure the hardware correctly at runtime.

by using a dts file, buildroot provides a standardized way to describe the hardware config of various devices and boards.
this allows for easier customization and porting of linux systems to different embedded platforms,
as the dts file can be modified or replaced to match the specific hardware requirements of a particular device or board.

<hr>

### # modify the dts using uboot cmd tool fdt
◆ background  
sometimes we want to test a driver with different dts setting,
one method is to rebuild the uboot or dtb (using dts) and flash to nor-flash.

sometimes the above method is bit overkill to some small modifications, alternatively,
we can modify the 'running config' of device-tree by uboot command "fdt", which is more efficient.

the following will introduce how to create or modify or delete device tree setting in uboot on brugal boards,
which should also be available as a ref for other boards.

◆ get the address of fdt
```text
=> bdinfo
boot_params = 0x0000000000000000
DRAM bank   = 0x0000000000000000
-> start    = 0x0000000400000000
-> size     = 0x0000000100000000
flashstart  = 0x0000000010000000
flashsize   = 0x0000000004000000
flashoffset = 0x0000000000000000
baudrate    = 115200 bps
relocaddr   = 0x00000004ffbf0000
reloc off   = 0x0000000000000000
Build       = 64-bit
current eth = dwc_xgemac
ethaddr     = 00:13:08:ce:11:00
IP addr     = <NULL>
fdt_blob    = 0x00000004ffb90e80
new_fdt     = 0x00000004ffb90e80
fdt_size    = 0x000000000002ef60
```
we could get the addr of fdt by new_fdt = 0x00000004ffb90e80.


◆ set fdt address
```text
=> fdt addr 4ffb90e80
```

◆ show all fdt commands
```text
=> fdt
fdt - flattened device tree utility commands

Usage:
fdt addr [-c]  <addr> [<length>]    - Set the [control] fdt location to <addr>
fdt move   <fdt> <newaddr> <length> - Copy the fdt to <addr> and make it active
fdt resize [<extrasize>]            - Resize fdt to size + padding to 4k addr + some optional <extrasize> if needed
fdt print  <path> [<prop>]          - Recursive print starting at <path>
fdt list   <path> [<prop>]          - Print one level starting at <path>
fdt get value <var> <path> <prop>   - Get <property> and store in <var>
fdt get name <var> <path> <index>   - Get name of node <index> and store in <var>
fdt get addr <var> <path> <prop>    - Get start address of <property> and store in <var>
fdt get size <var> <path> [<prop>]  - Get size of [<property>] or num nodes and store in <var>
fdt set    <path> <prop> [<val>]    - Set <property> [to <val>]
fdt mknode <path> <node>            - Create a new node after <path>
fdt rm     <path> [<prop>]          - Delete the node or <property>
fdt header [get <var> <member>]     - Display header info
                                      get - get header member <member> and store it in <var>
fdt bootcpu <id>                    - Set boot cpuid
fdt memory <addr> <size>            - Add/Update memory node
fdt rsvmem print                    - Show current mem reserves
fdt rsvmem add <addr> <size>        - Add a mem reserve
fdt rsvmem delete <index>           - Delete a mem reserves
fdt chosen [<start> <end>]          - Add/update the /chosen branch in the tree
                                        <start>/<end> - initrd start/end addr
NOTE: Dereference aliases by omitting the leading '/', e.g. fdt print ethernet0.
```

◆ list fdt
```text
=> fdt list /
/ {
        model = "DFMB-C";
        compatible = "nokia,dfmb-c";
        interrupt-parent = <0x00000001>;
        #address-cells = <0x00000002>;
        #size-cells = <0x00000002>;
        memory@0x400000000 {
        };
        reserved-memory {
        };
        config {
        };
        chosen {
        };
        cpus {
        };
```

```text
=> fdt list /cpld@0/misc@0
misc@0 {
        compatible = "nokia,cpld-misc";
        cpldinfo-name = "cdpu-cpld";
        uio-name = "cdpu-cpld";
        ga-over-regmap = <0x00000001>;
        regmap-name = "cdpu-cpld";
        version-reg = <0x00000006>;
        iteration-reg = <0x00000007>;
        compact-version-register-set;
        init-registers = <0x0000000a 0x00000000 0x0000001a 0x00000000 0x0000000c
                          0x00000000 0x0000001c 0x00000000 0x0000002f 0x00000000
                          0x00000030 0x00000000 0x00000031 0x00000000 0x00000032
                          0x00000000 0x00000033 0x00000000 0x00000034 0x00000000
                          0x00000035 0x00000000 0x00000036 0x00000000>;
};
```

◆ read fdt
```text
=> fdt get value var1 /cpld@0/misc@0 uio-name
=> fdt get size var2 /cpld@0/misc@0 uio-name

=> printenv
var1=cdpu-cpld
var2=0x0000000A
```

◆ modify fdt
```text
=> fdt set /cpld@0/misc@0 version-reg <0x10000006>

=> fdt list /cpld@0/misc@0
misc@0 {
        compatible = "nokia,cpld-misc";
        cpldinfo-name = "cdpu-cpld";
        uio-name = "cdpu-cpld";
        ga-over-regmap = <0x00000001>;
        regmap-name = "cdpu-cpld";
        version-reg = <0x10000006>;
```
after the fdt set clause, the value of version-reg was modified.

◆ create fdt
```text
=> fdt mknode /cpld@0 test@1

=> fdt list /cpld@0
cpld@0 {
        compatible = "alu,cpld", "simple-bus";
        regmap-name = "cdpu-cpld";
        test@1 {
        };
        misc@0 {
        };
        irqchip@10 {
        };

=> fdt list /cpld@0/test@1
test@1 {
};
```
as can be seen that an empty node of "test@1" was created.

now add some properties:
```text
=> fdt set /cpld@0/test@1 test-flag-no-value
=> fdt set /cpld@0/test@1 test-flag-string "string"
=> fdt set /cpld@0/test@1 test-flag-int <0x12345678>

=> fdt list /cpld@0/test@1
test@1 {
        test-flag-int = <0x12345678>;
        test-flag-string = "string";
        test-flag-no-value;
};
```

◆ delete fdt
```text
=> fdt rm /cpld@0/test@1 test-flag-no-value
test@1 {
        test-flag-int = <0x12345678>;
        test-flag-string = "string";
};
```

<hr>

### # code in action
todo: live examples needed
