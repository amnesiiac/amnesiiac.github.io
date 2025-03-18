---
layout: post
title: "udrv: udrv_ll_init -> drv module init (linux, udrv)"
author: "melon"
date: 1111-08-03 08:59
categories: "2024"
tags:
  - linux
  - driver
---

introduction to udrv ll_services.

<hr>

### # code walk through the main trunk of the udrv/driver model
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

2 udrv_drv_module_init: init driver backend data structure & operations.

```blurtext
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
    foreach_object_in_section(struct driver**, drv, controller_driver_section){     // foreach drv in controller section
        pr_debug("driver init: driver pointer: %p, name: %s\n", drv, (*drv)->name);
        ret = udrv_drv_register_driver(*drv);                                       // register controller drv on bus
        if(ret){
            return ret;
        }
    }
    foreach_object_in_section(struct driver**, drv, device_driver_section){         // for each drv in section
        pr_debug("driver init: driver pointer: %p, name: %s\n", drv, (*drv)->name);
        ret = udrv_drv_register_driver(*drv);                                       // register dev drv on bus
        if(ret){
            return ret;
        }
    }
    return 0;
}
```

3 register drv on the its claimed bus.

```blurtext
/* 3.1 udrv/base/driver.c */
int udrv_drv_register_driver(struct driver* drv){
    int ret = 0;
    if(!drv){
        pr_error("invalid arguments, func: %s, line: %d, drv: %p\n", __func__, __LINE__, drv);
        return -EINVAL;
    }
    if(drv->init){                                     // if driver has init cb
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

4 struct driver definition & basic driver operation definition.

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

<hr>

### # how to manage all the driver instance available under udrv?
1 define the helper macro for adding visible driver src (udrv/driver/driver_xxx.c) in 2 separate elf section:
the controller_driver_section and device_driver_section.

```blurtext
/* udrv/base/driver.h */
#define __controller_driver    section_attributes(controller_driver_section)
#define __device_driver        section_attributes(device_driver_section)

#define udrv_controller_driver(__driver) \
        static struct driver* __udrv_controller_##__driver __controller_driver = &(__driver)

#define udrv_device_driver(__driver) \
        static struct driver* __udrv_device_##__driver __device_driver = &(__driver)
```

the following list some existed controller drivers:

```blurtext
$(udrv project dir) grep -r 'udrv_controller_driver' .
./drivers/driver_misc_controller.c:udrv_controller_driver(misc_controller_driver);
./drivers/driver_chrdev_controller.c:udrv_controller_driver(chrdev_controller_driver);
./drivers/driver_interrupt_controller.c:udrv_controller_driver(interrupt_controller_driver);
./drivers/driver_spi_controller.c:udrv_controller_driver(spi_controller_driver);
./drivers/driver_file_controller.c:udrv_controller_driver(file_controller_driver);            // *
./drivers/driver_localbus_controller.c:udrv_controller_driver(localbus_controller_driver);
./drivers/driver_i2c_controller.c:udrv_controller_driver(i2c_controller_driver);
```

<p style="margin-bottom: 20px;"></p>

2 how to register a driver into customized elf section?  
a) take file_controller_driver as a case, the declaration of the driver src expansion is as:

```blurtext
/* udrv/drivers/driver_file_controller.c */
udrv_controller_driver(file_controller_driver)
```

take the macro definition at udrv/base/driver.h, the original declaration can be translated to:

```blurtext
static struct driver* __udrv_controller_file_controller_driver __controller_driver = &(file_controller_driver)
```

can be converted to the following form:

```blurtext
static struct driver* __udrv_controller_file_controller_driver section_attributes(controller_driver_section) \
= &(file_controller_driver)
```

take the section definition at udrv/utils/section.h into consideration, finally we get:

```blurtext
static struct driver* __udrv_controller_file_controller_driver                           \
                      __attribute__((unused, section("udrv_controller_driver_section"))) \
                      __attribute__((aligned(4)))= &file_controller_driver
```

b) udrv/drivers/driver_xxx.c macro definition expansion analysis:  
the \_\_attribute\_\_ is a gnu c mechanism to add characteristics to variables/function/types for the compiler.
the unused param is for avoiding compiler's unused variable warning, the section keyword is used to put
the pointer of the file driver into customized elf section: udrv_controller_driver_section.

c) purpose of aligning controller drivers & device drivers into customized section:  
sometimes we need this feature for convenience of iteration throughout the registered set of controller & device drivers.
placing each set of the drivers into customized section will make the iteration code happy (eliminate boring changes
to the for loop each time adding new driver src).

d) driver.h iteration part code analysis:  
the iteration works by navigating from the start address till the stop address of the defined section, with each step
size as the object_type passed in.
in the second snippet, we only need to declare the start & stop addr variable, the gcc (ld rather) will define
the exact value for the conspiracy variable between compiler & linker: \_\_start_##section_name & \_\_stop_##section_name.

```blurtext
/* udrv/base/driver.c */
section_range_marker_declare(controller_driver_section);  // declare start & stop addr for controller_driver_section
section_range_marker_declare(device_driver_section);      // declare start & stop addr for device_driver_section

/* udrv/utils/section.h: helper macro for the above declaration */
#define section_range_marker_declare(section_name)    \
        extern void* __start_udrv_##section_name;     \
        extern void* __stop_udrv_##section_name;

/* udrv/utils/section.h: iteration obj through the each section */
#define foreach_object_in_section(object_type, obj, section_name)    \       /* iterate over the range of section */
        for(obj = (object_type)section_range_start(section_name);    \       /* with a step = object_type */
            obj < (object_type)section_range_stop(section_name);     \
            obj += 1)
#endif
```

<p style="margin-bottom: 20px;"></p>

3 appendix code udrv/utils/section.h:

```blurtext
#ifndef __SECTION_H__
#define __SECTION_H__

#define __stringify_1(x...)     #x                                           /* convert params into literal */
#define __stringify(x...)       __stringify_1(x)                             /* convert wrapper name */

#define section_attributes(section_name)    \
        __attribute__((unused, section(__stringify(udrv_##section_name)))) __attribute__((aligned(4)))

#define section_range_marker_declare(section_name)    \
        extern void* __start_udrv_##section_name;    \                       /* add __start_udrv_xxx ptr def */
        extern void* __stop_udrv_##section_name;                             /* add __stop_udrv_xxx ptr def */

#define section_range_start(section_name)    &(__start_udrv_##section_name)  /* return addr of section start ptr */
#define section_range_stop(section_name)     &(__stop_udrv_##section_name)   /* return addr of section end ptr*/

#define foreach_object_in_section(object_type, obj, section_name)    \       /* iterate over the range of section */
        for(obj = (object_type)section_range_start(section_name);    \       /* with a step = object_type */
            obj < (object_type)section_range_stop(section_name);     \
            obj += 1)
#endif
```

<hr>

### # how does the specific driver of a dev register its default behavior & customized operations


<hr>

### # more details on the \_\_attribute\_\_ & section & __start/stop_SECTION trick
1 \_\_attribute\_\_: http://www.unixwiz.net/techtips/gnu-c-attributes.html

2 \_\_attribute\_\_ in gnu 4.0 doc:  
function attributes (see section): http://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Function-Attributes.html  
variable attributes (see section): http://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Variable-Attributes.html  
type attributes: http://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Type-Attributes.html

2 \_\_start/stop_##section_name:  
iteration usage template: https://stackoverflow.com/a/16552711/10515951  
a executable toy code of section iteration usage: https://stackoverflow.com/a/48550485/10515951
