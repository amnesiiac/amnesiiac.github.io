---
layout: post
title: "udrv: udrv_ll_init -> populate devices (linux, udrv)"
author: "melon"
date: 1111-08-05 20:19
categories: "2024"
tags:
  - linux
  - driver
---

introduction to udrv ll_services.

<hr>

### # code
udrv_ll_init: root udrv model initialization entrance.

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
    ret = udrv_bus_module_init();
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
    ret = udrv_populate_devices();                                            // *
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

udrv_populate_devices: recursively populate throughtout the abstract device model tree:
platform_bus => controller dev => devices (3 level), ref: udrv intro blog post.

```blurtext
/* 1 udrv/ll_services/udrv_basic_ll_services.c: low level services init */
static int udrv_populate_devices(){
    int ret;
    struct bus_type* bus;
    bus = udrv_bus_find_bus(UDRV_PLATFROM_BUS);                            // platform bus (the most parent) (2)
    if(!bus){
        pr_error("can not find platform-bus\n");
        return -ENOSYS;
    }
    ret = udrv_dev_populate_next_level_devices(NULL, bus);                 // populate controller devices (layer#1) (3)
    if(ret){
        pr_error("populate devices failed, error: %d\n", ret);
    }
    return ret;
}

/* 2 udrv/base/bus.c: */
struct bus_type* udrv_bus_find_bus(char* busname){                         // return bus obj by busname
    int i;
    for(i = 0; i < udrv_bus_number; i++){
        if(udrv_str_equals(udrv_buses[i]->name, busname)){
            return udrv_buses[i];
        }
    }
    return NULL;
}

/* 3 udrv/base/device.c: populate layer#2 device nodes */
int udrv_dev_populate_next_level_devices(struct device* parent, struct bus_type* bus){
    struct creating_device_context context = { parent, bus };
    devnode_t* node = parent ? parent->devnode : udrv_devnode_get_root();  // ref: dev module init (4)
    return udrv_devnode_foreach_object(                                    // ref: dev module init (3)
        node,
        udrv_dev_is_allowed_device_object,                                 // match_fn, ref: dev module init (5)
        udrv_dev_populate_device, &context                                 // handle_fn, (4)
    );
}

/* 3 udrv/base/device.c: */
struct creating_device_context {
    struct device* parent;
    struct bus_type* bus;
};
```

4 udrv_dev_populate_device: find device instance by devname, set dev->bus as the bus stored in context.

```blurtext
/* udrv/base/device.c: */
static int udrv_dev_populate_device(const char* name, devnode_t* node, void* data){
    struct device* dev;
    int ret = 0;
    struct creating_device_context* d = (struct creating_device_context*) data;
    if(!d->bus){
        pr_error("create device %s failed, not found the bus\n", name);
        return -EINVAL;
    }
    dev = udrv_dev_find_device_instance((char*) name);                     // return device obj by devname (4.1)
    if(!dev){
        pr_error("can not find device instance %s\n", name);
        return -ENODEV;
    }
    dev->bus = d->bus;                                                     // set all dev bus as current context bus
    ret = udrv_bus_add_device(dev);                                        // (5) register new dev to dev->bus
    if(ret){
        dev_error(dev, "add device: %s to bus: %s failed\n", dev->name, d->bus->name);
        udrv_dev_free_device(dev);
    }
    return ret;
}

/* 4.1 udrv/base/device.c  */
struct device* udrv_dev_find_device_instance(const char* devname){
    struct device* dev;
    if(!devname){
        return NULL;
    }
    list_for_each_entry(dev, &(device_instances), link_internal){          // (4.2)
        if(udrv_str_equals(dev->name, devname) || udrv_str_equals(dev->fullname, devname)){
            return dev;
        }
    }
    return NULL;
}

/* 4.2 udrv/utils/list.h: */
#define list_entry(link, type, member) \
    ((type *)((char *)(link)-(unsigned long)(&((type *)0)->member)))

#define list_head(list, type, member)       \
    list_entry((list)->next, type, member)

#define list_for_each_entry(pos, list, member)         \
    for(pos = list_head(list, typeof(*pos), member);   \
        &pos->member != (list);                        \
        pos = list_next(pos, member))
```

5 udrv_bus_add_device: add new dev obj attached to the bus.

```blurtext
/* udrv/base/bus.c */
int udrv_bus_add_device(struct device* dev){
    struct bus_type* bus;
    struct driver* drv = NULL;
    int ret = 0;
    if(!dev || !dev->bus){
        return -EINVAL;
    }
    if(!udrv_bus_are_all_dependency_devices_ready(dev)){                   // (6)
        dev_debug(dev, "add device: %s to defer init list\n", dev->name);
        udrv_bus_add_device_to_defer_init_list(dev);                       // (7)
        return 0;
    }
    bus = dev->bus;                                                        // get bus of the dev
    list_append(&(bus->device), &(dev->link));                             // append dev to bus->dev list
    if(list_empty(&(bus->driver))){                                        // if bus drv list is null, ret
        return 0;
    }
    udrv_bus_foreach_driver(bus, drv){                                     // populate the drv list registered in bus
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

/* 7 udrv/base/bus.c */
static int udrv_bus_are_all_dependency_devices_ready(struct device* dev){
    struct device* dep_dev;
    struct device_dependency* dep = NULL;
    if(list_empty(&(dev->link_dependencies))){                             // if cur dev is independant of all other dev
        return 1;                                                          // ready
    }
    list_for_each_entry(dep, &(dev->link_dependencies), link){             // for each of cur dev's dependancy
        dep_dev = udrv_devmap_find_device(dep->devname);
        if(!dep_dev || !udrv_dev_is_ready(dep_dev)){                       // if depdev no present on devmap or no ready
            return 0;                                                      // not ready
        }
    }
    return 1;
}

/* 6 udrv/base/bus.c */
static void udrv_bus_add_device_to_defer_init_list(struct device* dev){
    struct device* defer_dev;
    list_for_each_entry(defer_dev, &(defer_init_devices), link_defer){     // check if already in defer list
        if(defer_dev == dev){
            return;
        }
    }
    list_append(&(defer_init_devices), &(dev->link_defer));                // append cur dev into defer d-list
}
```
