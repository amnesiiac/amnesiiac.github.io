---
layout: post
title: "udrv: udrv_ll_init -> drv module init (linux, udrv)"
author: "melon"
date: 2024-08-03 08:59
categories: "2024"
tags:
  - linux
---

introduction to udrv ll_services.

<hr>

### # code
1 udrv_ll_init: root udrv model initialization entrance.

```text
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
    ret = udrv_drv_module_init();                                    // *
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

udrv_drv_module_init: init driver backend data structure & operations.

```text
/* udrv/base/driver.c */
int udrv_drv_module_init(void){
    int ret;
    ret = udrv_init_all_drivers();                // register all drivers, first the controller drvs, then the normal
    if(ret){                                      // the normal drv might has dependency on controller drvs.
        udrv_exit_all_drivers();
    }
    return ret;
}

/* udrv/base/driver.c */
static int udrv_init_all_drivers(void){
    struct driver** drv;
    int ret;
    foreach_object_in_section(struct driver**, drv, controller_driver_section){     // for each drv in controller section (utils)
        pr_debug("driver init: driver pointer: %p, name: %s\n", drv, (*drv)->name);
        ret = udrv_drv_register_driver(*drv);                                       // register controller drv on its bus
        if(ret){
            return ret;
        }
    }
    foreach_object_in_section(struct driver**, drv, device_driver_section){         // for each drv in section (utils)
        pr_debug("driver init: driver pointer: %p, name: %s\n", drv, (*drv)->name);
        ret = udrv_drv_register_driver(*drv);                                       // register dev drv on its bus
        if(ret){
            return ret;
        }
    }
    return 0;
}

/* udrv/base/driver.c: declare extern pointer of udrv controller drv section / device drv section */
section_range_marker_declare(controller_driver_section);  // extern void* __start/stop_udrv_controller_driver_section
section_range_marker_declare(device_driver_section);      // extern void* __start/stop_udrv_device_driver_section
```

regiseter drv on the claimed bus.

```text
/* 11.3 udrv/base/driver.c */
int udrv_drv_register_driver(struct driver* drv){
    int ret = 0;
    if(!drv){
        pr_error("invalid arguments, func: %s, line: %d, drv: %p\n", __func__, __LINE__, drv);
        return -EINVAL;
    }
    if(drv->init){                                     // if not init
        ret = drv->init(drv);                          // try call driver init method
        if(ret){
            drv_error(drv, "driver init failed\n");
            return ret;
        }
    }
    ret = udrv_bus_add_driver(drv);                    // register drv to the claimed bus, ref: bus-module-init
    if(ret){
        drv_error(drv, "add driver to bus failed\n");
    }
    return ret;
}
```

struct driver definition & basic driver operation definition.

```text
/* 11.1 udrv/base/driver.h */
struct driver {
    char*                               name;
    char*                               compatible;    // used to decide matched dev to a drv
    struct bus_type*                    bus;           // bus the drv is associated with
    struct list                         link;          // d-linked drv list used by bus for iteration
    int (*init)(struct driver* drv);                   // cb: when load drv or init drv
    int (*deinit)(struct driver* drv);                 // cb: when unload drv or deinit drv
    int (*probe)(struct device* dev);                  // cb: when drv try probe new dev (check compatibility)
    int (*remove)(struct device* dev);                 // cb: when dev matched with this drv is removed or detached
    struct drv_operation*               opt;           // operations to perform on matched dev
};

/* udrv/base/driver.h */
struct drv_operation {
    int (*read)(struct device* dev, void* args, void* data, int size);   // read from dev to buf (data)
    int (*write)(struct device* dev, void* args, void* data, int size);  // write from buf (data) to dev
    int (*xfer)(struct device* dev, void* args, void* tx_buf, int tx_len, void* rx_buf, int rx_len);  // transfer
};
```

utility functions: the section name symbols are typically defined by the linker script, consider the udrv
will be compiled as dynamic link lib libudrv.so (todo).

```text
/* udrv/utils/section.h */
#ifndef __SECTION_H__
#define __SECTION_H__

#define __stringify_1(x...)     #x                                 /* convert params into string literal */
#define __stringify(x...)       __stringify_1(x)                   /* convert wrapper name */

#define section_attributes(section_name)    \
        __attribute__((unused, section(__stringify(udrv_##section_name)))) __attribute__((aligned(4)))

#define section_range_marker_declare(section_name)    \
        extern void* __start_udrv_##section_name;    \             /* add __start_udrv_xxx ptr def */
        extern void* __stop_udrv_##section_name;                   /* add __stop_udrv_xxx ptr def */

#define section_range_start(section_name)    &(__start_udrv_##section_name)  /* return addr of section start ptr */
#define section_range_stop(section_name)     &(__stop_udrv_##section_name)   /* return addr of section end ptr*/

#define foreach_object_in_section(object_type, obj, section_name)    \       /* iterate over the range of section */
        for(obj = (object_type)section_range_start(section_name);    \       /* with a step = object_type */
             obj < (object_type)section_range_stop(section_name);    \
             obj += 1)
#endif
```
