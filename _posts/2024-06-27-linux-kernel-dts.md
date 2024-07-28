---
layout: post
title: "linux device tree source (linux, device, dts)"
author: "melon"
date: 2024-06-27 21:11
categories: "2024"
tags:
  - linux
---

device tree source (dts) file is used to describe the device tree model.
the dts file use tree structure to describe the device info like: num of cpu, mem base addr,
i2c subsystems info, spi subsystem info\...

```txt
                                           │            │
                                +──────────┤            │
                              +     ...    │            ├─────────────────+
                                +──────────┤            │  SDMMC controler  +
                                           │            ├─────────────────+
                     +─────────────────────┤            │
                   +     PCI controller    │            │       +      + 
                     +─────────────────────┤            │      + +    + +
                                           │ system bus │      │ │    │ │
                     +      +      +       │            │   spi│1│ spi│2│
                    + +    + +    + +      │            ├──────┴─┴────┴─┴──────+
                    │ │    │ │    │ │      │            │    SPI controller      +
                 i2c│1│ i2c│2│ i2c│3│      │            ├──────────────────────+
              +─────┴─┴────┴─┴────┴─┴──────┤            │
            +         i2c controller       │            │
              +────────────────────────────┤            ├─────────────────────────+
                                           │            │     GPIO controller       +
                                           │            ├─────────────────────────+
                                           │            │

```

the main trunk is system bus, with branches like: i2c controller, gpio controller,
spi controller\...
branches can also be separated, i.e. i2c branch has i2c1 and i2c2.

dts file is designed to describe the device layout info on the development board.

<hr>

### # why we need dts?
taking led driver as a instance, if want to change the gpio pin for led, then you should modify
the led driver src, recompile the driver and reload the driver.

however, the board with the same chip can rely on different hw resources, i.e. board-A choose
gpio-a for led, board-B choose grio-b. kernel gpio driver support both gpio-a and gpio-b,
then bus/device c code need to take care of the pin usage info.
as a result, the kernel device src get ugly and bloated.

by introducing dts info as a separate config, the device src code wont be botherred by the
annoying hardcode: when board is bringing up, uboot get executed to load kernel & dts files
into memory, then start the kernel with dts addr in mem pass to it.

check the /sys/firmware/devicetree:

<hr>

### # keyword
```txt
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 subject                           │ functionalities                                                             
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 device tree source file (dts)     │ describe device model info...
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 device tree binary file (dtb)     │ binary file used to migration linux system
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 device tree compiler (dtc)        │ compile dts file into dtb, located at scripts/dtc
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 device tree source include (dtsi) │ the header file of dts file to define common structures for reuse
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

<hr>

### # grammar of the dts file

<hr>

### # common nodes in dts

<hr>

### # dts usages

<hr>

### # modify the dts using uboot tool fdt
1 background  
sometimes there's need to test a driver with different dts settings,
one way is to rebuild the uboot or dtb (from dts) and flash to nor-flash.
however, the above method is bit overkill for testing small modifications, alternatively,
we can modify the running config of device-tree by uboot command fdt.

the following will introduce how to create & modify & delete device tree setting in uboot.

2 get the address of fdt
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


3 set fdt address
```text
=> fdt addr 4ffb90e80
```

4 show all fdt commands
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

5 list fdt
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

6 read fdt
```text
=> fdt get value var1 /cpld@0/misc@0 uio-name
=> fdt get size var2 /cpld@0/misc@0 uio-name

=> printenv
var1=cdpu-cpld
var2=0x0000000A
```

7 modify fdt
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

8 create fdt
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

9 delete fdt
```text
=> fdt rm /cpld@0/test@1 test-flag-no-value
test@1 {
        test-flag-int = <0x12345678>;
        test-flag-string = "string";
};
```

<hr>

### # kernel code for dts manuplation
todo
