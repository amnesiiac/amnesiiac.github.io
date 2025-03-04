---
layout: post
title: "udrv: udrv_ll_init -> dev module init (linux, udrv)"
author: "melon"
date: 1111-08-04 16:32
categories: "2024"
tags:
  - linux
  - driver
  - nokia
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
    ret = udrv_dev_module_init();                                     // *
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

udrv_dev_module_init: init device backend data structure & operations.

```blurtext
/* 1 udrv/base/device.c */
int udrv_dev_module_init(void){
    list_init(&device_instances);                                     // init device instance list
    return udrv_dev_create_all_device_instances();                    // (2)
}

/* 2 udrv/base/device.c */
static int udrv_dev_create_all_device_instances(){                    // create all available device instance
    struct creating_device_context context = { NULL, NULL };
    int ret = udrv_devnode_foreach_object(                            // foreach devnode matched, handle in turn (3)
        udrv_devnode_get_root(),                                      // get root node (5)
        udrv_dev_is_allowed_device_object,                            // match_fn: judge by json config (6)
        udrv_dev_create_device_instance,                              // handle_fn: foreach allowed, handle it (7)
        &context);
    return ret;
}
```

helper function to iterate over devnode data structures, and execute match_fn & handle_fn.

```blurtext
/* 3 udrv/base/devnode.c */
int udrv_devnode_foreach_object(devnode_t* base, int (*match_fn)(const char*, devnode_t*, void*),
                                int (*handle_fn)(const char*, devnode_t*, void*), void* data){
    int ret;
    devnode_t* node;
    const char* name;
    devnode_object_foreach(base, name, node){                         // (4)
        if(match_fn && !match_fn(name, node, data)){                  // if match_fn is set & not match, skip
            continue;
        }
        if(handle_fn && (ret = handle_fn(name, node, data))){         // if handle_fn is set, handle the node
            return ret;
        }
    }
    return 0;
}

/* 4 udrv/base/devnode.h */
#define devnode_object_foreach(devnode, key, value) \
        json_object_foreach(devnode, key, value)                      /* see jansson/src/jansson.h */

/* 5 udrv/base/devnode.c */
devnode_t* udrv_devnode_get_root(void){
    return json_root;
}
```

match_fn to judge whether a devnode is allowed to be created as device.

```blurtext
/* 6 udrv/base/device.c */
static int udrv_dev_is_allowed_device_object(const char* name, devnode_t* node, void* data){
    int is_allowed = 0;
    if(udrv_devnode_has_compatible(node)){                            // judge whether current dev node can be init
        is_allowed = udrv_dev_is_initialization_allowed(name);
        pr_debug("device: %s is %sallowed to do initialization\n", name, is_allowed ? "" : "NOT ");
    }
    return is_allowed;
}

/* udrv/base/devnode.c */
int udrv_devnode_has_compatible(devnode_t* node){                     // judge if json node has "compatible drv" info
    return json_object_get(node, "compatible") != NULL;
}

/* udrv/base/device.c */
static int udrv_dev_is_initialization_allowed(const char* devname){
    if(!devname){
        return 0;
    }
    if(device_filter_rule == UDRV_DEVICE_FILTER_NONE){                // no rule, all devices are allowed
        return 1;
    }
    if(device_filter_rule == UDRV_DEVICE_FILTER_WHITE_LIST){          // white list dev is allowed init
        return udrv_dev_is_in_filter_list(devname);
    }
    if(device_filter_rule == UDRV_DEVICE_FILTER_BLACK_LIST){          // black list dev is not allowed init
        return !udrv_dev_is_in_filter_list(devname);
    }
    return 1;                                                         // nonsense, but still ret allowed
}
```

handle_fn to create struct device instance of each matched devnode.

```blurtext
/* 7 udrv/base/device.c */
static int udrv_dev_create_device_instance(const char* name, devnode_t* node, void* data){
    struct device* dev;
    struct creating_device_context* d = (struct creating_device_context *) data;   // (7.1)
    struct creating_device_context context = { NULL, NULL };
    dev = udrv_dev_create_device(                                                  // create dev (8)
        d->parent, (char*) name,
        (char*) udrv_devnode_get_string(node, "compatible"),
        d->bus, node
    );
    if(!dev){
        pr_error("create device %s failed, maybe no memory\n", name);
        return -ENODEV;
    }
    list_append(&(device_instances), &(dev->link_internal));                       // append new dev obj to all dev list
    context.parent = dev;                                                          // set ctx.par as cur dev
    return udrv_devnode_foreach_object(node, udrv_dev_is_allowed_device_object,    // goon create its subnodes (3)
                                       udrv_dev_create_device_instance, &context);
}

/* 7.1 udrv/base/device.c */
struct creating_device_context {
    struct device*   parent;
    struct bus_type* bus;
};

/* 8 udrv/base/device.c */
struct device* udrv_dev_create_device(struct device* parent,
                                      char* name,
                                      char* compatible,
                                      struct bus_type* bus,
                                      devnode_t* devnode){
    struct device* dev = udrv_mem_zalloc(sizeof(*dev));                // alloc mem for deivce (9)
    if(!dev){
        return NULL;
    }
    dev->name = udrv_str_dup(name);                                    // fill in the dev field (9)
    dev->fullname = udrv_dev_generate_fullname(parent, dev);           // set dev full name
    dev->compatible = udrv_str_dup(compatible);                        // set compatible drv for dev
    dev->bus = bus;                                                    // set the bus attached to
    dev->parent = parent;                                              // set parent dev
    dev->devnode = devnode;
    dev->init_silence = udrv_dev_need_silence(devnode);
    dev->silence = dev->init_silence;
    dev->pluggable = udrv_dev_is_pluggable(devnode);                   // set pluggable
    udrv_dev_parse_capability(dev, devnode);
    dev->state = UDRV_DEVICE_STATE_CREATED;                            // set dev state

    list_init(&(dev->link));                                           // list node init for add to bus
    list_init(&(dev->link_internal));                                  // list node init for add to all dev obj list
    list_init(&(dev->link_defer));                                     // list node init for add to defer init list

    udrv_init_notification_chain(&(dev->event_notifications));         // init dev event notify list (10)
    udrv_init_notification_chain(&(dev->interrupt_notifications));     // init dev irq notify list (10)

    list_init(&(dev->link_dependencies));                              // init dev dep list node
    if(udrv_dev_parse_dependency(dev) < 0){                            // get & fill in dep list (8.1)
        udrv_dev_free_dependency(dev);
    }
    dev->lock = udrv_plt_lock_new();                                   // init dev lock (8.2)
    if(!dev->lock){
        udrv_dev_free_device(dev);
        dev = NULL;
    }
    if(dev){
        dev_debug(dev, "create device instance: %s done\n", dev->name);
    }
    return dev;
}

/* 8.1 udrv/base/device.c */
static int udrv_dev_parse_dependency(struct device* dev){
    devnode_t* dependency_node;
    devnode_t* dependency_nodes;
    int index;
    char* dependency_devname;
    struct device_dependency* dependency_dev;

    dependency_nodes = udrv_devnode_get_array_node(dev->devnode, "dependencies");
    if(!dependency_nodes){
        return 0;
    }
    devnode_array_foreach(dependency_nodes, index, dependency_node){
        if(!udrv_devnode_is_string(dependency_node)){
            continue;
        }
        dependency_dev = udrv_mem_zalloc(sizeof(*dependency_dev));
        if(!dependency_dev){
            dev_error(dev, "can not get memory for device dependency instance\n");
            return -ENOMEM;
        }
        dependency_devname = (char*)udrv_devnode_to_string(dependency_node);
        dependency_dev->devname = udrv_str_dup(dependency_devname);
        list_init(&(dependency_dev->link));
        list_append(&(dev->link_dependencies), &(dependency_dev->link));
        dev_info(dev, "add dependency device: %s to list\n", dependency_devname);
    }
    return 0;
}

/* udrv/platform/plt_adaptor.c */
struct udrv_lock* udrv_plt_lock_new(void){
    if(user_interfaces_ptr && user_interfaces_ptr->udrv_plt_lock_new_fn){  // use user-defined lock as prio
        return user_interfaces_ptr->udrv_plt_lock_new_fn();
    }
    return default_interfaces_ptr->udrv_plt_lock_new_fn();                 // fallback to default lock
}
```

struct device definition, core data structure of dev module.

```blurtext
/* 9 udrv/base/device.h */
struct device {
    char*                          name;
    char*                          fullname;
    char*i                         compatible;
    struct udrv_lock*              lock;                                  // lock for dev
    struct bus_type*               bus;                                   // dev's bus
    struct device*                 parent;                                // dev's parent (controller dev)
    struct list                    link_internal;                         // for internal management using
    struct list                    link;                                  // d-linked device list used for bus iteration
    struct list                    link_dependencies;                     // a list of dependency devices
    struct list                    link_defer;                            // defer init device d-linked list
    struct driver*                 drv;                                   // driver, ref: udrv-drv-init
    devnode_t*                     devnode;
    int                            capabilities;                          // dev caps
    int                            state;
    int                            events;
    struct udrv_notification_chain event_notifications;
    int                            nr_event_notifications;
    int                            nr_event_subscriber;
    struct udrv_notification_chain interrupt_notifications;               // todo
    int                            nr_interrupt_notifications;
    int                            nr_interrupt_subscriber;
    int                            init_silence;
    int                            silence;                               // do not print anything if set
    int                            pluggable;                             // hardware may be not inserted
    int                            locked;                                // lock state
    void*                          devdata;
    struct device_op_usage         usage[UDRV_USAGE_TYPE_MAX];
};

/* 10 udrv/utils/notification.c */
int udrv_init_notification_chain(struct udrv_notification_chain* chain){  // list of notifications
    if(!chain){
        return -EINVAL;
    }
    list_init(&(chain->notifications));                                   // init notification list head
    return 0;
}

/* udrv/utils/notification.h */
struct udrv_notification_chain {
    struct list notifications;
};
```
