---
layout: post
title: "udrv: udrv_ll_init -> bus module init (linux, udrv)"
author: "melon"
date: 1111-08-02 08:59
categories: "2024"
tags:
  - linux
  - driver
  - nokia
---

introduction to udrv ll_services.

<hr>

### # code
1 udrv_ll_init: root udrv model initialization entrance.

```blurtext
/* udrv/ll_services/udrv_basic_ll_services */
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
    ret = udrv_devnode_init(jsonfile);
    if(ret){
        pr_error("devnode module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_dev_init_device_filter();
    if(ret){
        pr_error("device filter init failed, error: %d\n", ret);
        return ret;
    }
    civret = udrv_ll_api_control_init();
    if(ret){
        pr_error("api control init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_ll_parse_configs_in_json();
    if(ret){
        pr_error("parse configs in json failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_ll_parse_arguments(argc, argv);
    if(ret){
        pr_error("parse arguments failed, error: %d\n", ret);
        return ret;
    }
    udrv_ll_apply_print_configs();
    ret = udrv_bus_module_init();                                     // *
    if(ret){
        pr_error("bus module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_drv_module_init();
    if(ret){
        pr_error("driver module init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_dev_module_init();
    if(ret){
        pr_error("set platform interfaces failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_populate_devices();
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


udrv_bus_module_init: init bus backend data structure & operations.

```blurtext
/* udrv/base/bus.c: buildup bus mode, init dev & drv placeholder for each bus type */
int udrv_bus_module_init(void){
    int i;
    for(i = 0; i < udrv_bus_number; i++){                  // for each bus type
        list_init(&(udrv_buses[i]->device));               // init bus->dev d-list
        list_init(&(udrv_buses[i]->driver));               // init bus->drv d-list
    }
    list_init(&defer_init_devices);                        // init static defer init devices
    return 0;
}
```

bus_type definition & static udrv supported bus type list initialization.

```blurtext
/* udrv/base/bus.c */
struct bus_type {
    char*       name;                                      // bus name
    struct list device;                                    // all device attached to this bus
    struct list driver;                                    // all driver registered to the bus
    int (*match)(struct device* dev, struct driver* drv);  // match between drv & dev under the bus
};

/* udrv/base/bus.c */
static int udrv_bus_number = sizeof(udrv_buses) / sizeof(udrv_buses[0]);
static struct bus_type* udrv_buses[] = {
    &udrv_platform_bus,
    &udrv_chrdev_bus,
    &udrv_interrupt_bus,
    &udrv_spi_bus,
    &udrv_i2c_bus,
    &udrv_i2c_smbus_bus,
    &udrv_file_bus,
    &udrv_localbus_bus,
    &udrv_misc_bus,
};
```

sub bus type definitions.

```blurtext
/* 10.4 udrv/base/bus.c */
static struct bus_type udrv_platform_bus = {
    .name = UDRV_PLATFROM_BUS,
    .match = udrv_bus_common_match,
};

static struct bus_type udrv_chrdev_bus = {
    .name = UDRV_CHRDEV_BUS,
    .match = udrv_bus_common_match,
};

static struct bus_type udrv_interrupt_bus = {
    .name = UDRV_INTRDEV_BUS,
    .match = udrv_bus_common_match,
};

static struct bus_type udrv_spi_bus = {
    .name = UDRV_SPI_BUS,
    .match = udrv_bus_common_match,
};

static struct bus_type udrv_i2c_bus = {
    .name = UDRV_I2C_BUS,
    .match = udrv_bus_common_match,
};

static struct bus_type udrv_i2c_smbus_bus = {
    .name = UDRV_I2C_SMBUS_BUS,
    .match = udrv_bus_common_match,
};

static struct bus_type udrv_file_bus = {
    .name = UDRV_FILE_BUS,
    .match = udrv_bus_common_match,
};

static struct bus_type udrv_localbus_bus = {
    .name = UDRV_LOCALBUS_BUS,
    .match = udrv_bus_common_match,
};

static struct bus_type udrv_misc_bus = {
    .name = UDRV_MISC_BUS,
    .match = udrv_bus_common_match,
};
```

common driver & device match method.

```blurtext
static int udrv_bus_common_match(struct device* dev, struct driver* drv){
    if(dev->compatible && drv->compatible){                         // for all valid l1 & l2 device
        return udrv_str_equals(dev->compatible, drv->compatible);   // if they compatible field the same -> matched
    }
    return 0;
}
```

register a new driver to its claimed bus.

```blurtext
/* 10.5 udrv/base/bus.c */
int udrv_bus_add_driver(struct driver* drv){
    struct bus_type* bus;
    struct device* dev = NULL;
    int ret = 0;
    if(!drv || !drv->bus){
        return -EINVAL;
    }
    bus = drv->bus;                                        // get the drv's claimed bus
    list_append(&(bus->driver), &(drv->link));             // append the new drv to the drv list known to bus
    if(list_empty(&(bus->device))){                        // no device under bus, return
        return 0;
    }
    udrv_bus_foreach_device(bus, dev){                     // for each dev matched with this drv under cur bus
        if(!bus->match(dev, drv)){
            continue;
        }
        dev->drv = drv;                                    // set dev->drv as cur drv if they matched each other
        ret = drv->probe(dev);                             // try use the new drv to probe all dev matched
        if(ret){
            pr_error("%s: driver %s probe device %s failed, bus %s\n", __func__, drv->name, dev->name, bus->name);
            break;
        }
        udrv_dev_set_state(dev, UDRV_DEVICE_STATE_READY);  // set dev ready
    }
    return ret;
}
```

attach a new device to its claimed bus.

```blurtext
/* udrv/base/bus.c */
int udrv_bus_add_device(struct device* dev){
    struct bus_type* bus;
    struct driver* drv = NULL;
    int ret = 0;
    if(!dev || !dev->bus){
        return -EINVAL;
    }
    if(!udrv_bus_are_all_dependency_devices_ready(dev)){                   // if dev's dependency dev are not ready
        dev_debug(dev, "add device: %s to defer init list\n", dev->name);
        udrv_bus_add_device_to_defer_init_list(dev);                       // add the dev into defer init list
        return 0;
    }
    bus = dev->bus;                                                        // get the dev claimed bus
    list_append(&(bus->device), &(dev->link));                             // add the dev to the bus's dev list
    if(list_empty(&(bus->driver))){
        return 0;
    }
    udrv_bus_foreach_driver(bus, drv){                                     // for each drv registered to the bus
        if(!bus->match(dev, drv)){                                         // if drv no match with the dev, skip
            continue;
        }
        dev->drv = drv;                                                    // if matched, set as the dev's drv
        ret = drv->probe(dev);                                             // try probe the new dev by all matched drv
        if(ret){
            pr_error("%s: driver %s probe device %s failed, bus %s\n", __func__, drv->name, dev->name, bus->name);
            break;
        }
        udrv_dev_set_state(dev, UDRV_DEVICE_STATE_READY);                  // set dev as ready state
    }
    return ret;
}

/* udrv/base/bus.c: check the dependency list of a dev are all ready */
static int udrv_bus_are_all_dependency_devices_ready(struct device* dev){
    struct device* dep_dev;
    struct device_dependency* dep = NULL;

    if(list_empty(&(dev->link_dependencies))){                  // dev has no dependency, return ready
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
```

add device to defer init list.

```blurtext
/* udrv/base/bus.c */
static struct list defer_init_devices;
static void udrv_bus_add_device_to_defer_init_list(struct device* dev){
    struct device* defer_dev;
    list_for_each_entry(defer_dev, &(defer_init_devices), link_defer){   // for each dev->link_defer in defered list
        if(defer_dev == dev){                                            // if dev already in the list, noop
            return;
        }
    }
    list_append(&(defer_init_devices), &(dev->link_defer));              // append dev->link_defer to defered list
}
```

helper macro definitions for the above add driver method.

```blurtext
/* udrv/base/bus.h: device & driver iteration helper function of bus */
#define udrv_bus_foreach_device(bus, dev) \
        list_for_each_entry(dev, &(bus->device), link)

#define udrv_bus_foreach_driver(bus, drv) \
        list_for_each_entry(drv, &(bus->driver), link)

/* utils/list.h */
#define list_for_each_entry(pos, list, member)          \
    for(pos = list_head(list, typeof(*pos), member);    \
         &pos->member != (list);                \
         pos = list_next(pos, member))
```
