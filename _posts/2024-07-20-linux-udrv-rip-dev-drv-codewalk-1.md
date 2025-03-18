---
layout: post
title: "udrv: rip device driver code walk I (linux, udrv)"
author: "melon"
date: 1111-07-20 10:46
categories: "2024"
tags:
  - linux
  - driver
---

rip device is hosted on eeprom chip, which is used to store non-volatile information of the product:
board-name, company-id, factory-id, factory-release-date, mac, serial num\...

on target board, the RIP info can be derived by fuse filesystem mounted on eeprom; while for simulation build,
no physical device existed, thus currently there's no virtual kdrv specified for RIP operations,
so we just use a linux file as backend for read/write to RIP.

rip code walk I mainly introduce the udrv main trunk design, taking rip dev testcase program as a case;
the rip code walk II will specialize on rip dev driver scope.

<hr>

### # code walk of udrv operations for rip device testcase
let's start from the upmost usecase script, go deeper till the bottom of the udrv, to get acknowledge of the simplest
storage device operation pipeline inside udrv project.

1 udrv/test/source/rip.c: main src to test read/write operations towards rip.

```blurtext
#include <string.h>

#include "debug.h"                                               // debug common interface
#include "udrv.h"                                                // udrv basic data structure
#include "udrv_ll_interfaces.h"                                  // low level interface
#include "udrv_rip.h"                                            // platform service interfaces

void udrv_test_rip(){                                            // rip operation proxied by udrv (test)
    int ret;
    char buf[512];                                               // placeholder
    char* devname = "rip";                                       // set the dev to operate
    char* fieldname = ".raw_full";                               // set the fieldname to read/write (todo)

    ret = udrv_read_rip(devname, fieldname, buf, 512);           // 3(a): read certain rip field to buf
    if(ret != 512){
        pr_error("read rip field '%s' failed, error: %d\n", fieldname, ret);
        return;
    }
    udrv_dbg_hexdump(buf, 512);                                  // dump & validate binary rip field info
}

int main(int argc, char* argv[]){
    int ret;
    struct udrv_init_config config = {                           // init udrv config with default interfaces
        .argc = argc,
        .argv = argv,
        .json_pathname = "udrv.json",                            // set target/simulation config json
        .interfaces = NULL
    };
    ret = udrv_init(&config);                                    // 2(a): buildup udrv backend data model by config
    if(ret){
        pr_error("udrv init faild, error: %d\n", ret);
        return ret;
    }
    udrv_test_rip();                                             // test operation on rip dev backend dev model
    return 0;
}
```

2 udrv/interfaces/udrv.h: define the data-driven config structure format for udrv, define the init
function for wholesome udrv model by json config data (udrv dts).

```blurtext
#ifndef __UDRV_H__
#define __UDRV_H__

#ifdef __cplusplus
extern "C" {
#endif

#include "plt_interfaces.h"

struct udrv_init_config {                        // key structure for data-driven initialized udrv config
    int                    argc;
    char**                 argv;
    char*                  json_pathname;
    struct plt_interfaces* interfaces;           // platform adapter interface
};

int udrv_init(struct udrv_init_config* config);  // 9.1: udrv ll data model & operations buildup by config

#ifdef __cplusplus
}
#endif
#endif
```

3 high-level device interfaces def & implementation (rip device).

```blurtext
/* 3(a) udrv/interfaces/udrv_rip.h: high-level device service interface definitions */
#ifndef __UDRV_RIP_H__
#define __UDRV_RIP_H__

#ifdef __cplusplus
extern "C" {
#endif

int udrv_read_rip(char* devname, char* fieldname, char* buf, int size);           // read rip field
int udrv_write_rip(char* devname, char* fieldname, char* buf, int size);          // write rip field

#ifdef __cplusplus
}
#endif
#endif
```

```blurtext
/* 3(b) udrv/services/udrv_rip.c: high-level service interface implementations */
#include <errno.h>

#include "udrv_rip.h"
#include "udrv_ll_interfaces.h"

static int udrv_read_write_rip(char* devname, char* fieldname, char* buf, int size, int is_write){
    int ret;
    struct device* dev;                                                           // init device
    DECLARE_RIP_DEV_ARGS(ripargs);                                                // 4: declare ripargs

    if(!devname || !fieldname || !buf || size < 1){
        pr_error("invalid arguments, func: %s, line: %d, devname: %p, fieldname: %p, buf: %p, size: %d\n",
                 __func__, __LINE__, devname, fieldname, buf, size);
        return -EINVAL;
    }
    ret = udrv_ll_dev_get_available_device(devname, &dev);                        // 5: get dev obj by name
    if(ret){
        pr_error("can not find device: %s, error: %d\n", devname, ret);
        return ret;
    }
    ripargs.fieldname = fieldname;                                                // set ripargs field for operate on
    if(is_write){
        ret = udrv_ll_dev_write(dev, &ripargs, buf, size);                        // 6: call low level write
    }
	else{
        ret = udrv_ll_dev_read(dev, &ripargs, buf, size);                         // 6: call low level read
    }
    if(ret < 0){
        dev_error(dev, "%s rip device failed, error: %d\n", is_write ? "write" : "read", ret);
    }
    return ret;
}

int udrv_read_rip(char* devname, char* fieldname, char* buf, int size){           // read
    return udrv_read_write_rip(devname, fieldname, buf, size, 0);
}

int udrv_write_rip(char* devname, char* fieldname, char* buf, int size){          // write
    return udrv_read_write_rip(devname, fieldname, buf, size, 1);
}
```

4 udrv/ll_interfaces/udrv_ll_driver_arguments.h: setup device args for registered operations.

```blurtext
/* 267,268 */
#define UDRV_GENERAL_OPERATION_DATA_IO_OF_FIELDNAME     0                  /* udrv operation on: field data io */
#define UDRV_GENERAL_OPERATION_DATA_IO_OF_RAW           1                  /* udrv operation on: raw data io */

/* 274,276 */
#define fieldname_dataio_members    const char* fieldname; int operation;  /* udrv dataio memeber: field & op */

/* 315,319 */
#define RIP_OPT_DATA_IO_OF_FIELDNAME    UDRV_GENERAL_OPERATION_DATA_IO_OF_FIELDNAME
struct rip_dev_args {
    fieldname_dataio_members
};
#define DECLARE_RIP_DEV_ARGS(args)    struct rip_dev_args args = {NULL, RIP_OPT_DATA_IO_OF_FIELDNAME}
```

5 udrv/ll_services/udrv_ll_basic_service.c: return dev obj matching the given name.

```blurtext
/* 5.1 udrv/ll_services/udrv_ll_basic_service.c */
int udrv_ll_dev_get_available_device(const char* devname, struct device** dev){
    struct device* dev_found;
    int ret;
    ret = udrv_ll_dev_get_device_instance(devname, &dev_found);     // 5.2: get dev obj by name from devmap
    if(ret){
        return ret;
    }
    if(!udrv_dev_is_ready(dev_found)){                              // check if the dev is ready
        return -ENOTREADY;
    }
    *dev = dev_found;
    return 0;
}

/* 5.2 udrv/ll_services/udrv_ll_basic_service.c */
int udrv_ll_dev_get_device_instance(const char* devname, struct device** dev){
    struct device* dev_found;
    if(!devname){
        return -EINVAL;
    }
    dev_found = udrv_devmap_find_device(devname);                   // 7: get dev by name in devmap
    if(!dev_found){
        return -ENODEV;
    }
    *dev = dev_found;
    return 0;
}
```

6 general io implementation by calling the low-level registered ops of the matched drv to the given dev.
for how does the dev->drv operation implemented & registered, please go next section for guidance.

```blurtext
/* 6.1 udrv/ll_services/udrv_ll_general_io_service.c */
int udrv_ll_dev_read(struct device* dev, void* args, void* data, int size){
    if(dev && dev->drv && dev->drv->opt && dev->drv->opt->read){   // dev exist, dev matched drv exist, drv has read opt
        return dev->drv->opt->read(dev, args, data, size);         // call read op of the dev's drv
    }
    pr_error("No implementation for the READ operation or no driver serves this device\n");
    return -EINVAL;
}

/* 6.2 udrv/ll_services/udrv_ll_general_io_service.c */
int udrv_ll_dev_write(struct device* dev, void* args, void* data, int size){
    if(dev && dev->drv && dev->drv->opt && dev->drv->opt->write){
        return dev->drv->opt->write(dev, args, data, size);        // call write op of dev's drv
    }
    pr_error("No implementation for the WRITE operation or no driver serves this device\n");
    return -EINVAL;
}
```

7 udrv/base/devmap.c: find device instance from devmap.

```blurtext
#include "devmap.h"
#include "device.h"

struct device* udrv_devmap_find_device(const char* devname){
    const char* dev_name = devname;
    struct device* dev;

    dev = udrv_dev_find_device_instance(dev_name);            // 8.1: find dev instance by devname
    if(!dev){                                                 // if devname not found, try find by alias name
        dev_name = udrv_devnode_get_alias_string(devname);
        dev = udrv_dev_find_device_instance(dev_name);
    }
    return dev;
}
```

8 find device obj by devname given by iterate over the dev list (list_for_each_entry helper macro).

```blurtext
/* 8.1 udrv/base/device.c  */
struct device* udrv_dev_find_device_instance(const char* devname){
    struct device* dev;
    if(!devname){
        return NULL;
    }
    list_for_each_entry(dev, &(device_instances), link_internal){
        if(udrv_str_equals(dev->name, devname) || udrv_str_equals(dev->fullname, devname)){
            return dev;
        }
    }
    return NULL;
}
```

```blurtext
/* 8.2 udrv/utils/list.h: list iteration helper macros */
#define list_entry(link, type, member) \                              /* return ptr to certain list entry */
    ((type *)((char *)(link)-(unsigned long)(&((type *)0)->member)))

#define list_head(list, type, member)       \                         /* return list head */
    list_entry((list)->next, type, member)

#define list_for_each_entry(pos, list, member)         \              /* iterate over each list entry */
    for(pos = list_head(list, typeof(*pos), member);   \
        &pos->member != (list);                        \
        pos = list_next(pos, member))
```

9 udrv services init process: 1) read user customized json configs (udrv.json); 2) buildup low level bus-drv-dev
model and other basic modules; 3) init udrv default services.

```blurtext
/* 9.1 udrv/services/udrv.c: application level services init, invoked by tests/rip.c */
#include <errno.h>

#include "utils.h"
#include "udrv_version.h"
#include "udrv.h"
#include "udrv_service.h"
#include "udrv_ll_interfaces.h"

int udrv_init(struct udrv_init_config* config){
    int ret;
    char* jsonfile;
    if(!config){
        pr_error("invalid argument, func: %s, line: %d, config: %p\n", __func__, __LINE__, config);
        return -EINVAL;
    }
    jsonfile = config->json_pathname;
    ret = udrv_file_access_ok(jsonfile);
    if(ret){
        pr_error("can not access the json config file: %s\n", jsonfile);
        return ret;
    }
    ret = udrv_ll_init(config->argc, config->argv, config->json_pathname, config->interfaces);  // 9.2: udrv ll init
    if(ret){
        pr_error("udrv low level init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_service_init();                                     // init default udrv services supported
    if(ret){
        pr_error("udrv services init failed, error: %d\n", ret);
        return ret;
    }
    return 0;
}
```
```blurtext
/* 9.2 udrv/ll_services/udrv_basic_ll_services.c: low level udrv device model init */
int udrv_ll_init(int argc, char* argv[], char* jsonfile, struct plt_interfaces* interfaces){
    int ret;
    if(!jsonfile){                                                   // read user config json (udrv.json)
        pr_error("json file was not given\n");
        return -EINVAL;
    }
    ret = udrv_get_jsonfiles_dir(jsonfile);
    if(ret){
        pr_error("get json file dir failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_plt_module_init(interfaces);                          // init platform module
    if(ret){
        pr_error("platform module failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_dbg_module_init();                                    // init debug module
    if(ret){
        pr_error("debug module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_devnode_init(jsonfile);                               // init devnode by user config json
    if(ret){
        pr_error("devnode module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_dev_init_device_filter();                             // init device filter rule
    if(ret){
        pr_error("device filter init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_ll_api_control_init();                                // init low-level api control module
    if(ret){
        pr_error("api control init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_ll_parse_configs_in_json();                           // parse config field in udrv.json
    if(ret){                                                         // e.g. print level, print verbose, api, dev filter
        pr_error("parse configs in json failed, error: %d\n", ret);  
        return ret;
    }
    ret = udrv_ll_parse_arguments(argc, argv);                       // parse udrv init cmdline argments
    if(ret){
        pr_error("parse arguments failed, error: %d\n", ret);
        return ret;
    }
    udrv_ll_apply_print_configs();                                   // apply cmdline/udrv.json configs
    ret = udrv_bus_module_init();                                    // init bus module (bus dev model)
    if(ret){
        pr_error("bus module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_drv_module_init();                                    // init drv module (drv dev model)
    if(ret){
        pr_error("driver module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_dev_module_init();                                    // init all available device data structure
    if(ret){
        pr_error("set platform interfaces failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_populate_devices();                                   // connect up the bus-drv-dev model
    if(ret){
        pr_error("populate devices failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_bus_add_all_defer_init_devices();                     // append all defer init dev to bus dev list
    if(ret){
        pr_error("populate defer init devices failed, error: %d\n", ret);
        return ret;
    }
    return 0;
}
```

```blurtext
/* 9.3 udrv/services/udrv_service.c: service for udrv sfp init */
int udrv_service_init(void){
    int ret;
    ret = udrv_service_sfp_init();                                   // udrv/services/udrv_sfp.c
    if(ret){
        pr_error("udrv sfp service init failed, error: %d\n", ret);
        return ret;
    }
    return 0;
}
```

<hr>

### # how the dev->drv->opt defined & registered in specific driver file?
remind the struct driver definition & basic driver operation definition in drv module intro post:

```blurtext
/* 4.1 udrv/base/driver.h */
struct driver {
    char*                               name;
    char*                               compatible;    // used to decide matched dev to a drv
    struct bus_type*                    bus;           // bus the drv is associated with
    struct list                         link;          // d-linked drv list used by bus for iteration
    int (*init)(struct driver* drv);                   // cb: when load drv or init drv
    int (*deinit)(struct driver* drv);                 // cb: when unload drv or deinit drv
    int (*probe)(struct device* dev);                  // cb: when drv try probe new dev (check compatibility)
    int (*remove)(struct device* dev);                 // cb: when dev matched with this drv is removed or detached
    struct drv_operation*               opt;           // 4.2: operations to perform on matched dev
};

/* 4.2 udrv/base/driver.h */
struct drv_operation {
    int (*read)(struct device* dev, void* args, void* data, int size);   // read from dev to buf (data)
    int (*write)(struct device* dev, void* args, void* data, int size);  // write from buf (data) to dev
    int (*xfer)(struct device* dev, void* args, void* tx_buf, int tx_len, void* rx_buf, int rx_len);  // transfer
};
```

firstly, we take a look at udrv.json config file:

```blurtext
/* udrv/tests/json/udrv.json */
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

    "file": {                                           // the layer1 is file controller drv
        "compatible": "udrv file controller",

        ...                                             // some other type of dev drv like: temp sensor...

        "rip0": {                                       // dev rip0, compatible with rip drv
            "compatible": "udrv rip device",
            "path": "vroot/rip"
        },

        "ntio-rip": {                                   // dev ntio-rip, compatible with rip drv
            "compatible": "udrv rip device",
            "following": "ntio",
            "cache-enabled": true,
            "path": "vroot/rip"
        },

        "rip1": {                                       // dev rip1, compatible with rip drv
            "compatible": "udrv rip device",
            "path": "vroot/rip1",
            "silence": true,
            "files": [
                "boardName",
                "mac"
            ]
        },
    },
    ...                                                 // other controller dev & its subdevs
}
```

thus, examine the rip driver code:

```blurtext
/* udrv/drivers/driver_rip.c */
static struct drv_operation rip_opts = {                // customized op: r & w
    .read = rip_read,                                   // implementation of r & w is in driver code, ref code walk II
    .write = rip_write,
};

static struct driver rip_driver = {                     // driver basic property & ops
    .name = "udrv rip driver",                          // name
    .compatible = "udrv rip device",                    // compatible field id
    .init = rip_init,                                   // cb for driver init to bus
    .probe = rip_probe,                                 // cb when probe new dev added to bus
    .opt = &rip_opts,                                   // setup customized ops to dev compatible with this drv
};

udrv_device_driver(rip_driver);
```

from which we can see the drv_operation (rip_read & rip_write) got registered as opts of rip_driver.
the following macro add this drv instance to udrv device driver section; after the udrv_ll_init finished,
all the above ll infra data/operation model are built finished.

finally in udrv/ll_services/udrv_ll_general_io_service.c, all io method invokation can be facilitated  by
dev->drv->opt; thus, in udrv/test/source/rip.c the upmost usecase script, the request to read from field
data from the rip dev got handled.
