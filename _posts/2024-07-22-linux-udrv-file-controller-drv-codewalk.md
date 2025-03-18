---
layout: post
title: "udrv: file controller driver code walk (linux, udrv)"
author: "melon"
date: 1111-07-22 10:36
categories: "2024"
tags:
  - linux
  - driver
---

file controller driver code walk will focus on the main trunk of file controller driver, which is widely used
in both target & simulation userspace platform apps for facilitating hardware access operations.

<hr>

### # code walk of controller driver of file device (from down to top)
1 driver basic struct definition & driver section setup:

```blurtext
/* udrv/drivers/driver_file_controller.c */
static struct driver file_controller_driver = {
    .name = "udrv file controller driver",              // drv name
    .compatible = "udrv file controller",               // for matching of dev <-> drv
    .init = file_controller_init,                       // cb func called right before this drv registered in udrv
    .probe = file_controller_probe,                     // cb func called when add new dev under drv's bus
    .opt = &file_controller_opts,                       // supported operations
};

/* udrv/drivers/driver_file_controller.c */
udrv_controller_driver(file_controller_driver);         // set file controller driver into controller driver section
```

<p style="margin-bottom: 20px;"></p>

2 init function invoked when load the file controller driver into controller driver section:

```blurtext
/* udrv/drivers/driver_file_controller.c */
static int file_controller_init(struct driver* drv){
    drv->bus = udrv_bus_find_bus(UDRV_PLATFROM_BUS);              // find & set file controller drv->bus as platform bus
    return 0;
}
```

<p style="margin-bottom: 20px;"></p>

3 probe function invoked when add new abstract dev to file controller driver's bus:

```blurtext
static int file_controller_probe(struct device* dev){
    int ret = 0;
    struct bus_type* bus = udrv_bus_find_bus(UDRV_FILE_BUS);      // find & return ptr to udrv file bus
    ret = udrv_dev_populate_next_level_devices(dev, bus);         // populate next level dev, set each dev->bus
    return ret;                                                   // next level dev = all dev controlled by this drv
}
```

<p style="margin-bottom: 20px;"></p>

4 file controller supported operation registration.

```blurtext
static struct drv_operation file_controller_opts = {              // register supported operation to drv
    .read = file_controller_read,                                 // read
    .write = file_controller_write,                               // write
};
```

4.1 read & write wrapper implementations.

```blurtext
/* udrv/drivers/driver_file_controller.c */
static int file_controller_read(struct device* dev, void* args, void* data, int size){
    return file_controller_read_write(dev, args, data, size, 0);
}

/* udrv/drivers/driver_file_controller.c */
static int file_controller_write(struct device* dev, void* args, void* data, int size){
    return file_controller_read_write(dev, args, data, size, 1);
}
```

4.2 read & write backend implementations.

```blurtext
static int file_controller_read_write(struct device* dev, void* args, void* data, int size, int is_write){
    int ret;
    int fd;
    struct file_controller_args* fileargs = (struct file_controller_args*) args;   // reformat caller args
    if(!dev || !args || !data || size < 1){                                        // exit with err for invalid args
        pr_error("invalid arguments, func: %s, line: %d, dev: %p, args: %p, data: %p, size: %d\n",
                 __func__, __LINE__, dev, args, data, size);
        return -EINVAL;
    }
    if(fileargs->fd < 0 && !fileargs->pathname){                                   // if no valid fd & no valid pathname
        pr_error("invalid arguments, func: %s, line: %d, fd: %d, pathname: %p\n",  // report err
                 __func__, __LINE__, fileargs->fd, fileargs->pathname);
        return -EINVAL;
    }
    fd = fileargs->fd;
    if(fd < 0){                                                      // if no valid fd, try operate on pathname
        if(is_write){                                                // get fd for the file with fpathname for write
            fd = udrv_file_open_writeonly(fileargs->pathname);
        }
        else{                                                        // get fd for the file with pathname for read
            fd = udrv_file_open_readonly(fileargs->pathname);
        }
        if(fd < 0){
            dev_error(dev, "can not open file: %s, error: %d\n", fileargs->pathname, fd);
            return fd;
        }
    }
    ret = udrv_file_lseek(fd, fileargs->offset);                     // seek the result by fd + offset
    if(ret < 0){
        dev_error(dev, "can not lseek file, error: %d\n", ret);
        return ret;
    }
    if(is_write){
        if(fileargs->should_truncate){                               // if the file to write need truncate
            ret = udrv_file_truncate(fd, fileargs->offset);          // 4.3.1 truncate f to offset length
            if(ret < 0){
                dev_warn(dev, "truncate file failed, fd: %d, length: %d, error: %d\n", fd, fileargs->offset, ret);
            }
        }
        ret = udrv_file_write(fd, data, size);                       // 4.3.2 write to fd with data of size
        if(ret > 0){
            udrv_file_sync(fd);                                      // 4.3.3 fsync for flush data to disk
        }
        else{
            dev_error(dev, "write file failed, fd: %d, error: %d\n", fd, ret);
        }
    }
    else{
        ret = udrv_file_read(fd, data, size);                        // 4.3.4 read data of size from fd
        if(ret < 0){
            dev_error(dev, "read file failed, fd: %d, error: %d\n", fd, ret);
        }
    }
    udrv_file_close(fd);
    return ret;
}
```

4.3 udrv low level utility apis for file related operations.

```blurtext
/* 4.3.1 udrv/utils/utils.c */
int udrv_file_truncate(int fd, int length){
    int ret = ftruncate(fd, length);
    ret = ret < 0 ? -errno : 0;
    return ret;
}

// the write() system call does not necessarily write the amount of requested bytes, it can write less.
// moreover, all system calls in linux can be interrupted by a signal, in which case EINTR is returned
// and the system call should simply be restarted.

/* 4.3.2 udrv/utils/utils.c */
int udrv_file_write(int fd, void* buf, int size){
    int ret = 0;
    int len = size;
    while((ret = write(fd, buf, len)) != 0){
        if(ret == -1){
            if(errno == EINTR){                           // if write is intrrupted, we just restart it
                continue;                                 // the write return the byte written, but here wont care?
            }
            if(errno == EAGAIN || errno == EWOULDBLOCK){  // if write to non-blocking fd, will get EAGAIN or EWOULDBLOCK
                break;                                    // so just let the udrv_write api return with msg to caller
            }
            ret = -errno;
            break;
        }
        break;
    }
    return ret;
}

/* 4.3.3 udrv/utils/utils.c */
int udrv_file_sync(int fd){
    return fsync(fd);
}
```

<hr>

### # udrv file controller driver config usecase

```blurtext
{
    "configs" : {
        "print_level": "info",
        "print_verbose": "enable",

        "enable_api": [
            "udrv_read_cpld_raw_data",
            "udrv_write_cpld_raw_data"
        ],

        "device_filter": {
            "rule": "blacklist",
            "list": ["temp3"]
        }
    },

    "file": {
        "compatible": "udrv file controller",

        "temp0": {
            "compatible": "udrv temp sensor device",
            "hwmon": true,
            "path": "vroot/temps/temp0"
        },
        "temp1": {
            "compatible": "udrv temp sensor device",
            "hwmon": true,
            "median_value": true,
            "median_value_interval": "500",
            "path": "vroot/temps/temp1"
        },
        "temp2": {
            "compatible": "udrv temp sensor device",
            "path": "vroot/temps/temp2"
        },
        "temp3": {
            "compatible": "udrv temp sensor device",
            "path": "vroot/temps/temp3"
        },
        "rip0": {
            "compatible": "udrv rip device",
            "path": "vroot/rip"
        },
        "ntio-rip": {
            "compatible": "udrv rip device",
            "following": "ntio",
            "cache-enabled": true,
            "path": "vroot/rip"
        },
        "rip1": {
            "compatible": "udrv rip device",
            "path": "vroot/rip1",
            "silence": true,
            "files": [
                "boardName",
                "mac"
            ]
        },
        "intr_storm": {
            "compatible": "udrv regular file device",
            "path": "vroot/regular"
        },
        "regular0": {
            "compatible": "udrv regular file device",
            "path": "vroot/rip"
        },

        "regular1": {
            "compatible": "udrv regular file device",
            "path": "vroot/rip",
            "readonly": true
        },
        "regular2": {
            "compatible": "udrv regular file device",
            "path": "vroot/regular",
            "readonly": true
        },
        "regular3": {
            "compatible": "udrv regular file device",
            "path": "vroot/sfp_extregs",
            "value-map":[
                { "filename": "alarm", "direction": "to-file",   "memory": "0", "file": "clear" },
                { "filename": "alarm", "direction": "to-memory", "memory": "0", "file": "clear" },
                { "filename": "alarm",                           "memory": "1", "file": "raise"  }
            ],
            "truncate_on_write": true
        },
        "fan0": {
            "compatible": "udrv fan device",
            "path": "vroot/sys/class/hwmon/hwmon6/device"
        },
        "fan1": {
            "compatible": "udrv fan device",
            "path": "vroot/sys/class/hwmon/hwmon8/device"
        },
        "ina220": {
            "compatible": "udrv sysfs ina2xx device",
            "median_value": true,
            "sample_interval": "500",
            "retry" : {"delay_ms": 500, "max_count": 3},
            "correction": {
                "in1_input": {
                    "multiple": 12,
                    "intercept": 5,
                    "divisor": 10
                },
                "power1_input": {
                    "multiple": 12,
                    "intercept": 5,
                    "divisor": 10
                }
            },
            "path": "vroot/sys/class/hwmon/hwmon0/device"
        },
        "i2c0": {
            "compatible": "udrv file backend i2c device",
            "path": "vroot/i2c/i2c-0.data",
            "bitfields": {
                "f1": {"offset":"0x00", "start":"0x00", "bits":"8"},
                "f2": {"offset":"0x00", "start":"0x00", "bits":"4"},
                "f3": {"offset":"0x00", "start":"0x04", "bits":"4"}
            }
        },
        "sfp0": {
            "compatible": "udrv file backend sfp device",
            "dependencies": ["file_backend_cpld"],
            "following": "ntio",
            "banks": [
                "vroot/sfp/sfp_bank0.data",
                "vroot/sfp/sfp_bank1.data"
            ],
            "events": {
                "hotplug": {
                    "intrdev":             "intr_sfp0_presence",
                    "reg":                 "cpld.sfp0_presence",
                    "privatedata":         "0x1"
                },
                "rx_los": {
                    "intrdev":             "intr_sfp0_rx_los",
                    "reg":                 "cpld.sfp0_rx_los",
                    "privatedata":         "0x1",
                    "occur":               "0x40",
                    "recov":               "0x80"
                },
                "tx_fault": {
                    "intrdev":             "intr_sfp0_tx_fault",
                    "reg":                 "cpld.sfp0_tx_fault",
                    "privatedata":         "0x1",
                    "occur_recov_profile": "sfp_tx_fault"
                },
                "tx_state": {
                    "reg": "cpld.sfp0_tx_state",
                    "privatedata": "0x1"
                },
                "poweron": {
                    "reg": "cpld.sfp0_poweron",
                    "privatedata": "0x1"
                },
                "alarm": {
                    "reg": "regular3.alarm",
                    "privatedata": "0x1"
                }
            },
            "extended-registers": {
                "device": "sfp0-extregs"
            }
        },

        "sfp1": {
            "compatible": "udrv file backend sfp device",
            "banks": [
                "vroot/sfp/sfp_bank0.data",
                "vroot/sfp/sfp_bank1.data"
            ],
            "forced-ready": true
        },
        "sfp3rd": {
            "compatible": "udrv file backend sfp device",
            "banks": [
                "vroot/sfp/RTXM167-522.data",
                "vroot/sfp/sfp_bank1.data"
            ],
            "forced-ready": true
        },
        "sfp3rd_sc": {
            "compatible": "udrv file backend sfp device",
            "banks": [
                "vroot/sfp/PTB38J0-6538E-SC.data",
                "vroot/sfp/sfp_bank1.data"
            ],
            "forced-ready": true
        },
        "sfp3rd_32d": {
            "compatible": "udrv file backend sfp device",
            "banks": [
                "vroot/sfp/SFP-GBX20-32D.data",
                "vroot/sfp/sfp_bank1.data"
            ],
            "forced-ready": true
        },
        "sysfs_cpld": {
            "compatible": "udrv sysfs cpld device",
            "path": "vroot/sys/class/cpld"
        },
        "gpiodev": {
            "compatible" : "udrv sysfs gpio device",
            "path" : "vroot/sys/class/gpio",
            "chip" : "gpiochip0",
            "silence" : true,
            "ports" : {
                "port0": {"index": "0", "direction": "in"},
                "port1": {"index": "1", "direction": "out"},
                "port2": {"index": "2", "direction": "modifiable"},
                "port3": {"index": "3", "direction": "default"}
            }
        },
        "remoteproc": {
            "compatible" : "udrv sysfs remoteproc device",
            "path" : "vroot/sys/class/remoteproc/remoteproc0/state"
        },
        "pcf8574_0": {
            "compatible": "udrv file backend pcf8574 device",
            "initial_value": "0xf5",
            "bitfields": {
                "portall":  {"offset":"0x00", "start":"0x00", "bits":"8"},
                "port0":    {"offset":"0x00", "start":"0x00", "bits":"1"},
                "port1":    {"offset":"0x00", "start":"0x01", "bits":"1"},
                "port2":    {"offset":"0x00", "start":"0x02", "bits":"1"},
                "port3":    {"offset":"0x00", "start":"0x03", "bits":"1"},
                "port4":    {"offset":"0x00", "start":"0x04", "bits":"1"},
                "port5":    {"offset":"0x00", "start":"0x05", "bits":"1"},
                "port6":    {"offset":"0x00", "start":"0x06", "bits":"1"},
                "port7":    {"offset":"0x00", "start":"0x07", "bits":"1"}
            }
        },
        "import-leds": "led.json",
        "import-cpld": "cpld.json"
    },
}
```
