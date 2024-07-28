---
layout: post
title: "udrv: RIP EEPROM device code walk (linux, udrv)"
author: "melon"
date: 2024-07-20 10:46
categories: "2024"
tags:
  - linux
  - todo
---

RIP device is hosted on EEPROM chip, which is used to store non-volatile infomation of the product:
board-name, company-id, factory-id, factory-release-date, mac, serial num\....

on target board, the RIP info can be derived by fuse filesystem mounted on EEPROM; while on simulation
framework, no physical device existed, thus currently there's no virtual kdrv specified for RIP operations,
so we just use a linux file as backend for read/write to RIP.

<hr>

### # udrv operations for udrv testcase code walk
1 udrv/test/source/rip.c: main src to test read/write operations towards rip.

```text
#include <string.h>

#include "debug.h"                                      // debug common interface
#include "udrv.h"                                       // udrv basic data structure
#include "udrv_ll_interfaces.h"                         // low level interface
#include "udrv_rip.h"                                   // platform service interfaces

void udrv_test_rip(){                                   // rip operation proxied by udrv (test)
    int ret;
    char buf[512];                                      // placeholder
    char* devname = "rip";                              // set the dev to operate
    char* fieldname = ".raw_full";                      // set the fieldname to read/write (todo)

    ret = udrv_read_rip(devname, fieldname, buf, 512);  // see 3(a)
    if(ret != 512){
        pr_error("read rip field '%s' failed, error: %d\n", fieldname, ret);
        return;
    }

    udrv_dbg_hexdump(buf, 512);                         // validate binary file info
}

int main(int argc, char* argv[]){
    int ret;
    struct udrv_init_config config = {                  // init udrv config with default interfaces
        .argc = argc,
        .argv = argv,
        .json_pathname = "udrv.json",                   // set target/simulation config json
        .interfaces = NULL
    };
    ret = udrv_init(&config);                           // see 2(a), init platform interface by default
    if(ret){
        pr_error("udrv init faild, error: %d\n", ret);
        return ret;
    }
    udrv_test_rip();
    return 0;
}
```

2 udrv/interfaces/udrv.h: define the data-driven config structure format for udrv, define the initialize
function for wholesome udrv model by config data.

```text
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

int udrv_init(struct udrv_init_config* config);  // see (9.1), udrv config initializer

#ifdef __cplusplus
}
#endif
#endif
```

3 high-level device interfaces def & implementation (rip device).

```text
/* 3(a) udrv/interfaces/udrv_rip.h: high-level device service interface definitions */
#ifndef __UDRV_RIP_H__
#define __UDRV_RIP_H__

#ifdef __cplusplus
extern "C" {
#endif

int udrv_read_rip(char* devname, char* fieldname, char* buf, int size);   // read rip info by fieldname
int udrv_write_rip(char* devname, char* fieldname, char* buf, int size);  // write rip info by fieldname

#ifdef __cplusplus
}
#endif
#endif


/* 3(b) udrv/services/udrv_rip.c: high-level service interface implementations */
#include <errno.h>

#include "udrv_rip.h"
#include "udrv_ll_interfaces.h"

static int udrv_read_write_rip(char* devname, char* fieldname, char* buf, int size, int is_write){
    int ret;
    struct device* dev;                                                   // init device
    DECLARE_RIP_DEV_ARGS(ripargs);                                        // see (4), init rip dev args

    if(!devname || !fieldname || !buf || size < 1){
        pr_error("invalid arguments, func: %s, line: %d, devname: %p, fieldname: %p, buf: %p, size: %d\n",
                 __func__, __LINE__, devname, fieldname, buf, size);
        return -EINVAL;
    }
    ret = udrv_ll_dev_get_available_device(devname, &dev);                // see (5) get dev obj by name
    if(ret){
        pr_error("can not find device: %s, error: %d\n", devname, ret);
        return ret;
    }
    ripargs.fieldname = fieldname;                                        // setup rip dev args fieldname
    if(is_write){
        ret = udrv_ll_dev_write(dev, &ripargs, buf, size);                // see (6), low level write
    }
	else{
        ret = udrv_ll_dev_read(dev, &ripargs, buf, size);                 // see (6), low level read
    }
    if(ret < 0){
        dev_error(dev, "%s rip device failed, error: %d\n", is_write ? "write" : "read", ret);
    }
    return ret;
}

int udrv_read_rip(char* devname, char* fieldname, char* buf, int size){   // itf
    return udrv_read_write_rip(devname, fieldname, buf, size, 0);
}

int udrv_write_rip(char* devname, char* fieldname, char* buf, int size){  // itf
    return udrv_read_write_rip(devname, fieldname, buf, size, 1);
}
```

4 udrv/ll_interfaces/udrv_ll_driver_arguments.h: setup device args for registered operations.

```text
/* 267,268 */
#define UDRV_GENERAL_OPERATION_DATA_IO_OF_FIELDNAME     0   /* data io by fieldname */
#define UDRV_GENERAL_OPERATION_DATA_IO_OF_RAW           1   /* data io by raw data */

...

/* 274,276 */
#define fieldname_dataio_members    const char* fieldname; int operation;

...

/* 315,319 */
#define RIP_OPT_DATA_IO_OF_FIELDNAME    UDRV_GENERAL_OPERATION_DATA_IO_OF_FIELDNAME
struct rip_dev_args {
    fieldname_dataio_members
};
#define DECLARE_RIP_DEV_ARGS(args)    struct rip_dev_args args = {NULL, RIP_OPT_DATA_IO_OF_FIELDNAME}
```

5 udrv/ll_services/udrv_ll_basic_service.c: return dev matching the given name.

```text
/* udrv/ll_services/udrv_ll_basic_service.c */
int udrv_ll_dev_get_available_device(const char* devname, struct device** dev){
    struct device* dev_found;
    int ret;
    ret = udrv_ll_dev_get_device_instance(devname, &dev_found);
    if(ret){
        return ret;
    }
    if(!udrv_dev_is_ready(dev_found)){
        return -ENOTREADY;
    }
    *dev = dev_found;
    return 0;
}

int udrv_ll_dev_get_device_instance(const char* devname, struct device** dev){
    struct device* dev_found;
    if(!devname){
        return -EINVAL;
    }
    dev_found = udrv_devmap_find_device(devname);      // see (7), get device by devmap (return addr of device)
    if(!dev_found){
        return -ENODEV;
    }
    *dev = dev_found;
    return 0;
}
```

6 udrv/ll_services/udrv_ll_general_io_service.c: low-level io interface implementations by invoking
the registered operations from a matched driver of given device.

```text
/* 6.1 udrv/ll_services/udrv_ll_general_io_service.c */
int udrv_ll_dev_read(struct device* dev, void* args, void* data, int size){
    if(dev && dev->drv && dev->drv->opt && dev->drv->opt->read){
        return dev->drv->opt->read(dev, args, data, size);   // read op of rip dev matched drv
    }
    pr_error("No implementation for the READ operation or no driver serves this device\n");
    return -EINVAL;
}

/* 6.2 udrv/ll_services/udrv_ll_general_io_service.c */
int udrv_ll_dev_write(struct device* dev, void* args, void* data, int size){
    if(dev && dev->drv && dev->drv->opt && dev->drv->opt->write){
        return dev->drv->opt->write(dev, args, data, size);  // write op of rip dev matched drv
    }
    pr_error("No implementation for the WRITE operation or no driver serves this device\n");
    return -EINVAL;
}
```

7 udrv/base/devmap.c: find device instance from devmap.

```text
#include "devmap.h"
#include "device.h"

struct device* udrv_devmap_find_device(const char* devname){
    const char* dev_name = devname;
    struct device* dev;

    dev = udrv_dev_find_device_instance(dev_name);            // get dev instance by devname, see (8.1)
    if(!dev){                                                 // if devname not found, try find by alias name
        dev_name = udrv_devnode_get_alias_string(devname);
        dev = udrv_dev_find_device_instance(dev_name);
    }
    return dev;
}
```

8 udrv/base/device.c: device related definitions & initialization ops & info expose ops.

```text
/* 8.1 udrv/base/device.c  */
struct device* udrv_dev_find_device_instance(const char* devname){
    struct device* dev;
    if(!devname){
        return NULL;
    }
    // todo ???????
    list_for_each_entry(dev, &(device_instances), link_internal){
        if(udrv_str_equals(dev->name, devname) || udrv_str_equals(dev->fullname, devname)){
            return dev;
        }
    }
    return NULL;
}

/* 8.2 udrv/utils/list.h: todo cursor illustraion added here */
#define list_entry(link, type, member) \
    ((type *)((char *)(link)-(unsigned long)(&((type *)0)->member)))

#define list_head(list, type, member)       \
    list_entry((list)->next, type, member)

#define list_for_each_entry(pos, list, member)         \
    for(pos = list_head(list, typeof(*pos), member);   \
        &pos->member != (list);                        \
        pos = list_next(pos, member))
```

9 when does the low level devices info get registered? and buildup the device list?

```text
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
    jsonfile = config->json_pathname;       // board specific udrv.json access for key struct
    ret = udrv_file_access_ok(jsonfile);
    if(ret){
        pr_error("can not access the json config file: %s\n", jsonfile);
        return ret;
    }
    ret = udrv_ll_init(config->argc, config->argv, config->json_pathname, config->interfaces);  // see (9.2)
    if(ret){
        pr_error("udrv low level init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_service_init();              // register udrv services hooks for upper hardware access apps
    if(ret){
        pr_error("udrv services init failed, error: %d\n", ret);
        return ret;
    }
    return 0;
}

/* 9.2 udrv/ll_services/udrv_basic_ll_services.c: low level services init */
int udrv_ll_init(int argc, char* argv[], char* jsonfile, struct plt_interfaces* interfaces){
    int ret;
    if(!jsonfile){
        pr_error("json file was not given\n");
        return -EINVAL;
    }
    ret = udrv_get_jsonfiles_dir(jsonfile);
    if(ret){
        pr_error("get json file dir failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_plt_module_init(interfaces);
    if(ret){
        pr_error("platform module failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_dbg_module_init();
    if(ret){
        pr_error("debug module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_devnode_init(jsonfile);       // init devnode by json config input
    if(ret){
        pr_error("devnode module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_dev_init_device_filter();     // init device filter rule
    if(ret){
        pr_error("device filter init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_ll_api_control_init();
    if(ret){
        pr_error("api control init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_ll_parse_configs_in_json();   // parse udrv config: printlevel, devnode, device filter...
    if(ret){
        pr_error("parse configs in json failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_ll_parse_arguments(argc, argv);
    if(ret){
        pr_error("parse arguments failed, error: %d\n", ret);
        return ret;
    }
    udrv_ll_apply_print_configs();           // set udrv print level
    ret = udrv_bus_module_init();            // see (10): init all bus data structure
    if(ret){
        pr_error("bus module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_drv_module_init();            // see (11): init all driver data structure
    if(ret){
        pr_error("driver module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_dev_module_init();            // see (12): init all available device data structure
    if(ret){
        pr_error("set platform interfaces failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_populate_devices();           // see (13.1): further buildup the bus-drv-dev model
    if(ret){
        pr_error("populate devices failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_bus_add_all_defer_init_devices();
    if(ret){
        pr_error("populate defer init devices failed, error: %d\n", ret);
        return ret;
    }
    return 0;
}
```

xx: devnode init
```text
/* udrv/base/devnode.c */
int udrv_devnode_init(char* json_pathname){
    int ret;
    char* json_real_pathname;
    if(!json_pathname){
        pr_error("invalid argument, json_pathname: %p\n", json_pathname);
        return -EINVAL;
    }
    ret = udrv_path_realpath(json_pathname, &json_real_pathname);
    if(ret){
        pr_error("can not get json file realpath, error: %d\n", ret);
        return ret;
    }
    ret = udrv_devnode_load_all_jsons(json_real_pathname);           // see ()
    udrv_str_free(json_real_pathname);
    if(ret){
        pr_error("load json failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_devnode_check();                                      // see ()
    if(ret){
        pr_error("precheck json config failed, error: %d\n", ret);
        return ret;
    }
    return 0;
}

/* udrv/base/devnode.c */
static int udrv_devnode_load_all_jsons(char* json_pathname){
    int ret;
    char* json_dir;
    devnode_t* node = udrv_devnode_load_json(json_pathname);         // see ()
    if(!node){
        pr_error("load mail json failed\n");
        return -EINVAL;
    }
    /* get json file dir for loading sub json file(s) */
    ret = udrv_path_dirname(json_pathname, &json_dir);
    if(ret){
        pr_error("get json file dir failed, error: %d\n", ret);
        return ret;
    }
    udrv_devnode_load_subjsons(json_dir, node);
    /* save json node entry */
    json_root = node;

    return 0;
}
```


10: init bus backend data structure.

```text
/* 10.1 udrv/base/bus.c: buildup bus mode, init dev & drv placeholder for each bus type */
static int udrv_bus_number = sizeof(udrv_buses) / sizeof(udrv_buses[0]);
int udrv_bus_module_init(void){
    int i;
    for(i = 0; i < udrv_bus_number; i++){                  // for each bus type
        list_init(&(udrv_buses[i]->device));               // init bus->dev d-list
        list_init(&(udrv_buses[i]->driver));               // init bus->drv d-list
    }
    list_init(&defer_init_devices);
    return 0;
}

/* 10.2 udrv/base/bus.c */
struct bus_type {
    char*        name;                                     // bus name
    struct list  device;                                   // all device attached to this bus
    struct list  driver;                                   // all driver known to the bus
    int (*match)(struct device* dev, struct driver* drv);  // match a drv & dev
};

/* 10.3 udrv/base/bus.c */
static int udrv_bus_number = sizeof(udrv_buses) / sizeof(udrv_buses[0]);
static struct bus_type* udrv_buses[] = {
    &udrv_platform_bus,                          // see (10.4) sub bus type structure def
    &udrv_chrdev_bus,
    &udrv_interrupt_bus,
    &udrv_spi_bus,
    &udrv_i2c_bus,
    &udrv_i2c_smbus_bus,
    &udrv_file_bus,
    &udrv_localbus_bus,
    &udrv_misc_bus,
};

/* 10.4 udrv/base/bus.c: i.e for platform bus, with match function registered */
static struct bus_type udrv_platform_bus = {
    .name = UDRV_PLATFROM_BUS,
    .match = udrv_bus_common_match,
};

static int udrv_bus_common_match(struct device* dev, struct driver* drv){
    if(dev->compatible && drv->compatible){      // dev & drv should compatible with each other for a match
        return udrv_str_equals(dev->compatible, drv->compatible);
    }
    return 0;
}

/* 10.5 udrv/base/bus.c: add a driver to its bus, probe the all dev matched with under the bus */
int udrv_bus_add_driver(struct driver* drv){
    struct bus_type* bus;
    struct device* dev = NULL;
    int ret = 0;

    if(!drv || !drv->bus){
        return -EINVAL;
    }
    bus = drv->bus;                             // get cur drv's bus
    list_append(&(bus->driver), &(drv->link));  // append new drv series after the existed drv list known to bus
    if(list_empty(&(bus->device))){             // if no device under bus
        return 0;
    }
    udrv_bus_foreach_device(bus, dev){          // see (10.6) for each dev matched with this drv
        if(!bus->match(dev, drv)){
            continue;
        }
        dev->drv = drv;                         // set dev->drv as cur drv if they matched each other
        ret = drv->probe(dev);                  // let drv probe the dev
        if(ret){
            pr_error("%s: driver %s probe device %s failed, bus %s\n", __func__, drv->name, dev->name, bus->name);
            break;
        }
        udrv_dev_set_state(dev, UDRV_DEVICE_STATE_READY);  // set dev ready
    }
    return ret;
}

/* 10.6 udrv/base/bus.h: device & driver iteration helper function of bus */
#define udrv_bus_foreach_device(bus, dev) \
        list_for_each_entry(dev, &(bus->device), link)

#define udrv_bus_foreach_driver(bus, drv) \
        list_for_each_entry(drv, &(bus->driver), link)

/* utils/list.h */
#define list_for_each_entry(pos, list, member)          \
    for (pos = list_head(list, typeof(*pos), member);   \
         &pos->member != (list);                \
         pos = list_next(pos, member))
```

11: init udrv driver backend data structure.

```text
/* 11.1 udrv/base/driver.c */
int udrv_drv_module_init(void){
    int ret;
    ret = udrv_init_all_drivers();                  // see (11.2)
    if(ret){
        udrv_exit_all_drivers();
    }
    return ret;
}

/* 11.2 udrv/base/driver.c */
static int udrv_init_all_drivers(void){
    struct driver** drv;
    int ret;
    // init controller dev drv
    foreach_object_in_section(struct driver**, drv, controller_driver_section){
        pr_debug("driver init: driver pointer: %p, name: %s\n", drv, (*drv)->name);
        ret = udrv_drv_register_driver(*drv);       // see (11.3), register controller drv on its bus
        if(ret){
            return ret;
        }
    }
    // init dev drv (might depend on controller drv)
    foreach_object_in_section(struct driver**, drv, device_driver_section){
        pr_debug("driver init: driver pointer: %p, name: %s\n", drv, (*drv)->name);
        ret = udrv_drv_register_driver(*drv);       // see (11.3), register dev drv on its bus
        if(ret){
            return ret;
        }
    }
    return 0;
}

/* 11.3 udrv/base/driver.c: register drv to bus interface */
int udrv_drv_register_driver(struct driver* drv){
    int ret = 0;
    if(!drv){
        pr_error("invalid arguments, func: %s, line: %d, drv: %p\n", __func__, __LINE__, drv);
        return -EINVAL;
    }
    if(drv->init){
        ret = drv->init(drv);
        if(ret){
            drv_error(drv, "driver init failed\n");
            return ret;
        }
    }
    ret = udrv_bus_add_driver(drv);                    // see (10.5), add drv to its bus
    if(ret){
        drv_error(drv, "add driver to bus failed\n");
    }
    return ret;
}

/* 11.1 udrv/base/driver.h */
struct driver {
    char*                               name;
    char*                               compatible;    // used to decide matched dev to a drv
    struct bus_type*                    bus;           // bus the drv is associated with
    struct list                         link;          // d-linked drv list used by bus for iteration
    int (*init)(struct driver* drv);                   // callback when drv is loaded or init
    int (*deinit)(struct driver* drv);                 // callback when drv is unloaded or deinit
    int (*probe)(struct device* dev);                  // callback when dev is detected & compatible with the drv
    int (*remove)(struct device* dev);                 // callback when dev managed by drv is removed or detached
    struct drv_operation*               opt;           // see (11.2)
};

/* 11.2 udrv/base/driver.h: the actual lowest level operation to interact with device */
struct drv_operation {
    int (*read)(struct device* dev, void* args, void* data, int size);   // read from dev to buf (data)
    int (*write)(struct device* dev, void* args, void* data, int size);  // write from buf (data) to dev
    int (*xfer)(struct device* dev, void* args, void* tx_buf, int tx_len, void* rx_buf, int rx_len);  // rx/tx dev data transfer
};
```

12: init udrv device backend data structure by given config json.

```text
/* 12.1 udrv/base/device.c */
int udrv_dev_module_init(void){
    list_init(&device_instances);
    return udrv_dev_create_all_device_instances();             // see 12.2
}

/* 12.2 udrv/base/device.c */
static int udrv_dev_create_all_device_instances(){             // create all available device instance
    struct creating_device_context context = { NULL, NULL };
    int ret =  udrv_devnode_foreach_object(                    // see (12.3)
                   udrv_devnode_get_root(),                    // see (12.4)
                   udrv_dev_is_allowed_device_object,          // see (12.5) match_fn: judge by json config
                   udrv_dev_create_device_instance,            // see (12.6) handle_fn: foreach allowed, handle it
                   &context);
    return ret;
}

/* 12.3 udrv/base/devnode.c */
int udrv_devnode_foreach_object(devnode_t* base, int (*match_fn)(const char*, devnode_t*, void*),
                                int (*handle_fn)(const char*, devnode_t*, void*), void* data){
    int ret;
    devnode_t* node;
    const char* name;
    devnode_object_foreach(base, name, node){                  // see (12.4) macro
        if(match_fn && !match_fn(name, node, data)){           // if match_fn is set & not match, skip
            continue;
        }
        if(handle_fn && (ret = handle_fn(name, node, data))){  // if handle_fn is set, handle the node
            return ret;
        }
    }
    return 0;
}

/* 12.4 udrv/base/devnode.c */
devnode_t* udrv_devnode_get_root(void){
    return json_root;
}

/* udrv/base/devnode.h */
#define devnode_object_foreach(devnode, key, value) \
        json_object_foreach(devnode, key, value)               /* see jansson/src/jansson.h */

/* 12.5 udrv/base/device.c */
static int udrv_dev_is_allowed_device_object(const char* name, devnode_t* node, void* data){
    int is_allowed = 0;
    if(udrv_devnode_has_compatible(node)){                     // judge whether current dev node can be init
        is_allowed = udrv_dev_is_initialization_allowed(name);
        pr_debug("device: %s is %sallowed to do initialization\n", name, is_allowed ? "" : "NOT ");
    }
    return is_allowed;
}

/* udrv/base/devnode.c */
int udrv_devnode_has_compatible(devnode_t* node){              // judge if json node has "compatible drv" reigist
    return json_object_get(node, "compatible") != NULL;
}

/* udrv/base/device.c */
static int udrv_dev_is_initialization_allowed(const char* devname){
    if(!devname){
        return 0;
    }
    if(device_filter_rule == UDRV_DEVICE_FILTER_NONE){         // no rule, all devices are allowed
        return 1;
    }
    if(device_filter_rule == UDRV_DEVICE_FILTER_WHITE_LIST){   // white list dev is allowed init
        return udrv_dev_is_in_filter_list(devname);
    }
    if(device_filter_rule == UDRV_DEVICE_FILTER_BLACK_LIST){   // black list dev is not allowed init
        return !udrv_dev_is_in_filter_list(devname);
    }
    return 1;                                                  // nonsense, but still ret allowed
}

/* 12.6 udrv/base/device.c */
static int udrv_dev_create_device_instance(const char* name, devnode_t* node, void* data){
    struct device* dev;
    struct creating_device_context* d = (struct creating_device_context *) data;
    struct creating_device_context context = { NULL, NULL };
    dev = udrv_dev_create_device(                              // see 12.7 create dev by compatible info parsed
        d->parent, (char*) name,
        (char*) udrv_devnode_get_string(node, "compatible"),
        d->bus, node
    );
    if(!dev){
        pr_error("create device %s failed, maybe no memory\n", name);
        return -ENODEV;
    }
    list_append(&(device_instances), &(dev->link_internal));
    context.parent = dev;
    return udrv_devnode_foreach_object(node, udrv_dev_is_allowed_device_object,
                                       udrv_dev_create_device_instance, &context);
}

/* 12.7 udrv/base/device.c: device key structure initialization helper */
struct device* udrv_dev_create_device(struct device* parent,
                                      char* name,
                                      char* compatible,
                                      struct bus_type* bus,
                                      devnode_t* devnode){
    struct device* dev = udrv_mem_zalloc(sizeof(*dev));         // see (12.8) alloc mem for deivce
    if(!dev){
        return NULL;
    }
    dev->name = udrv_str_dup(name);                             // see (12.8) fill in the dev field
    dev->fullname = udrv_dev_generate_fullname(parent, dev);    // set dev full name
    dev->compatible = udrv_str_dup(compatible);                 // set compatible drv for dev
    dev->bus = bus;                                             // set the bus attached to
    dev->parent = parent;                                       // set parent dev
    dev->devnode = devnode;
    dev->init_silence = udrv_dev_need_silence(devnode);
    dev->silence = dev->init_silence;
    dev->pluggable = udrv_dev_is_pluggable(devnode);            // set pluggable
    udrv_dev_parse_capability(dev, devnode);
    dev->state = UDRV_DEVICE_STATE_CREATED;                     // set dev state

    list_init(&(dev->link));                                    // init dev lists
    list_init(&(dev->link_internal));
    list_init(&(dev->link_defer));
    udrv_init_notification_chain(&(dev->event_notifications));     // see (12.9), init dev event notify list
    udrv_init_notification_chain(&(dev->interrupt_notifications)); // see (12.9), init dev irq notify list

    list_init(&(dev->link_dependencies));                       // init dev dependancy
    if(udrv_dev_parse_dependency(dev) < 0){
        udrv_dev_free_dependency(dev);
    }
    dev->lock = udrv_plt_lock_new();                            // init dev lock
    if(!dev->lock){
        udrv_dev_free_device(dev);
        dev = NULL;
    }
    if(dev){
        dev_debug(dev, "create device instance: %s done\n", dev->name);
    }
    return dev;
}

/* 12.8 udrv/base/device.h: device key structure definition */
struct device {
    char*                          name;
    char*                          fullname;
    char*i                         compatible;
    struct udrv_lock*              lock;                        // ...
    struct bus_type*               bus;                         // bus
    struct device*                 parent;
    struct list                    link_internal;               // for internal management using
    struct list                    link;                        // d-linked device list used for bus iteration
    struct list                    link_dependencies;           // a list of dependency devices
    struct list                    link_defer;                  // defer init device d-linked list
    struct driver*                 drv;                         // driver, see (11)
    devnode_t*                     devnode;
    int                            capabilities;
    int                            state;
    int                            events;
    struct udrv_notification_chain event_notifications;
    int                            nr_event_notifications;
    int                            nr_event_subscriber;
    struct udrv_notification_chain interrupt_notifications;
    int                            nr_interrupt_notifications;
    int                            nr_interrupt_subscriber;
    int                            init_silence;
    int                            silence;                     // do not print anything if set
    int                            pluggable;                   // hardware may be not inserted
    int                            locked;                      // lock state
    void*                          devdata;
    struct device_op_usage         usage[UDRV_USAGE_TYPE_MAX];
};

/* 12.9 udrv/utils/notification.c */
int udrv_init_notification_chain(struct udrv_notification_chain* chain){  // list of notifications
    if(!chain){
        return -EINVAL;
    }
    list_init(&(chain->notifications));                         // init notification list head
    return 0;
}

/* udrv/utils/notification.h */
struct udrv_notification_chain {
    struct list notifications;
};
```

13: populate devices and initialize layered bus-drv-dev model from the most parent node: platform bus,
then comes the layer#1 model: the controller abstract devices, finally comes the layer#2
model: the concrete device that at service.

ref: udrv's two-layer device management model structure in blog post: linux-udrv.

```text
/* 13.1 udrv/ll_services/udrv_basic_ll_services.c: low level services init */
static int udrv_populate_devices(){
    int ret;
    struct bus_type* bus;
    bus = udrv_bus_find_bus(UDRV_PLATFROM_BUS);                    // see (13.2) platform bus (the most parent bus)
    if(!bus){
        pr_error("can not find platform-bus\n");
        return -ENOSYS;
    }
    ret = udrv_dev_populate_next_level_devices(NULL, bus);         // see (13.3) populate controller devices (layer#1)
    if(ret){
        pr_error("populate devices failed, error: %d\n", ret);
    }
    return ret;
}

/* 13.2 udrv/base/bus.c: return bus by busname */
struct bus_type* udrv_bus_find_bus(char* busname){
    int i;
    for(i = 0; i < udrv_bus_number; i++){
        if(udrv_str_equals(udrv_buses[i]->name, busname)){
            return udrv_buses[i];
        }
	}
    return NULL;
}

/* 13.3 udrv/base/device.c: populate layer#2 device nodes */
int udrv_dev_populate_next_level_devices(struct device* parent, struct bus_type* bus){
    struct creating_device_context context = { parent, bus };
    devnode_t* node = parent ? parent->devnode : udrv_devnode_get_root();  // see (12.4)
    return udrv_devnode_foreach_object(                                    // see (12.3) ...
        node,
        udrv_dev_is_allowed_device_object,                                 // see (12.5)
        udrv_dev_populate_device, &context                                 // see (13.4)
    );
}

/* 13.3 udrv/base/device.c: */
struct creating_device_context {
    struct device* parent;
    struct bus_type* bus;
};

/* 13.4 udrv/base/device.c: */
static int udrv_dev_populate_device(const char* name, devnode_t* node, void* data){
    struct device* dev;
    int ret = 0;
    struct creating_device_context* d = (struct creating_device_context*) data;
    if(!d->bus){
        pr_error("create device %s failed, not found the bus\n", name);
        return -EINVAL;
    }
    dev = udrv_dev_find_device_instance((char*) name);             // see (8.1) return device obj by devname
    if(!dev){
        pr_error("can not find device instance %s\n", name);
        return -ENODEV;
    }
    dev->bus = d->bus;                                             // set all dev bus as current context bus
    ret = udrv_bus_add_device(dev);                                // see (13.5), 
    if(ret){
        dev_error(dev, "add device: %s to bus: %s failed\n", dev->name, d->bus->name);
        udrv_dev_free_device(dev);
    }
    return ret;
}

/* 13.5 udrv/base/bus.c */
int udrv_bus_add_device(struct device* dev){
    struct bus_type* bus;
    struct driver* drv = NULL;
    int ret = 0;
    if(!dev || !dev->bus){
        return -EINVAL;
    }
    if(!udrv_bus_are_all_dependency_devices_ready(dev)){                   // see (13.6)
        dev_debug(dev, "add device: %s to defer init list\n", dev->name);
        udrv_bus_add_device_to_defer_init_list(dev);                       // see (13.7)
        return 0;
    }
    bus = dev->bus;                                                        // get bus of the dev
    list_append(&(bus->device), &(dev->link));                             // append dev to bus->dev list
    if(list_empty(&(bus->driver))){                                        // if bus drv list is null, ret
        return 0;
    }
    udrv_bus_foreach_driver(bus, drv){                                     // populate bus drv list
        if(!bus->match(dev, drv)){                                         // cur dev not match with cur drv, skip
            continue;
        }
        dev->drv = drv;                                                    // set cur drv as cur dev's drv
        ret = drv->probe(dev);                                             // invoke cur drv probe function for init
        if(ret){
            pr_error("%s: driver %s probe device %s failed, bus %s\n", __func__, drv->name, dev->name, bus->name);
            break;
        }
        udrv_dev_set_state(dev, UDRV_DEVICE_STATE_READY);                  // set cur dev state as ready
    }
    return ret;
}

/* 13.7 udrv/base/bus.c */
static int udrv_bus_are_all_dependency_devices_ready(struct device* dev){
    struct device* dep_dev;
    struct device_dependency* dep = NULL;

    if(list_empty(&(dev->link_dependencies))){                  // if cur dev is independant of all other dev, return ready
        return 1;
    }
    list_for_each_entry(dep, &(dev->link_dependencies), link){  // for each of cur dev's dependancy
        dep_dev = udrv_devmap_find_device(dep->devname);
        if(!dep_dev || !udrv_dev_is_ready(dep_dev)){            // if dep-dev not present on devmap or not ready
            return 0;
        }
    }
    return 1;
}

/* 13.6 udrv/base/bus.c */
static void udrv_bus_add_device_to_defer_init_list(struct device* dev){
    struct device* defer_dev;
    list_for_each_entry(defer_dev, &(defer_init_devices), link_defer){  // check if already in defer list
        if(defer_dev == dev){
            return;
        }
    }
    list_append(&(defer_init_devices), &(dev->link_defer));             // append cur dev into defer d-list
}
```





utility: doubly linked list operations.

```text
/* udrv/utils/list.h: init empty doubly-linked list */
static inline void list_init(struct list* list){
    list->next = list;
    list->prev = list;
}

/* udrv/utils/list.h: insert new_link list ahead of link list of original d-linked list */
static inline void list_insert(struct list* link, struct list* new_link){
    new_link->prev = link->prev;
    new_link->next = link;
    new_link->prev->next = new_link;
    new_link->next->prev = new_link;
}
```


todo: separate the device --- driver --- bus model abstraction build up for a single article.
todo: add necessary dev drv bus model for rip dev as a simplest case.
todo: add necessary irq or hotplug code if possible
