---
layout: post
title: "udrv: uio interrupt device driver code walk (linux, udrv)"
author: "melon"
date: 1111-07-24 22:44
categories: "2024"
tags:
  - linux
  - driver
  - nokia
  - todo
---

this article will focus on the main trunk of the uio interrupt device driver driver, which is widely used in
both target & emulation userspace platform apps for facilitating hardware access operations.

<hr>

### # code walk of controller driver of interrupt device (from down to top)
1 driver basic struct definition & driver section setup:

```blurtext
/* udrv/drivers/driver_uio_interrupt.c */
static struct driver uio_interrupt_driver = {
    .name = "udrv uio interrupt driver",                      // drv name
    .compatible = "udrv uio interrupt device",                // for matching between dev <-> drv
    .init = uio_intr_init,                                    // cb func called right before this drv registered in udrv
    .probe = uio_intr_probe,                                  // cb func called when add new dev under this drv's bus
    .opt = &uio_intr_opts,                                    // supported operations
};

/* udrv/drivers/driver_uio_interrupt.c */
udrv_device_driver(uio_interrupt_driver);                     // set uio irq drv into udrv device driver section
```

<p style="margin-bottom: 20px;"></p>

2 uio interrupt device backend data structure definitions (the devdata):

```blurtext
/* udrv/drivers/driver_uio_interrupt.c */
struct uio_intr_devdata {
    char*                          uioname;
    struct udrv_uio_info*          uioinfo;                                          // 2.1
    int                            auto_enable;
    unsigned int                   missed_nr_interrupts;
    unsigned int                   nr_interrupts;
    int                            monitor_type;
    struct udrv_notification_chain sub_intr_notifications;                           // 2.2
    struct udrv_kvmap*             uioinfo_kvmap;                                    // 2.3
    struct interrupt_regs_context* regs_context;
};
```

```blurtext
/* 2.1 udrv/platform/plt_adapter.c */
struct udrv_uio_info {                                                               // udrv plt uio info def
    char*          uioname;                                                          // uio dev name
    int            fd;                                                               // ?
    void*          vaddr;                                                            // ?
    int            size;                                                             // ?
};

/* udrv/ll_services/udrv_ll_interrupt_service.c */
struct interrupt_regs_context {                                                      // irq reg config ctx
    struct device* dev;                                                              // owner dev
    struct list    regfields;                                                        // reg field info lst
};
```

```blurtext
/* 2.2 udrv/utils/notification.h */
struct udrv_notification_chain {                                                     // all notifications to cur dev
    struct list notifications;                                                       // notification list
};
```

```blurtext
/* 2.3 udrv/utils/kvmap.c */
struct udrv_kvmap {                                                                  // ?
    struct list elements;
    int (*key_compare_fn)(const void* key1, const void* key2);
    void (*free_value_fn)(void* value);
};
```

```blurtext
/* udrv/drivers/driver_uio_interrupt.c */
struct invoke_data {                                                                 // args of dev irq trigger cb func
    struct device*        dev;                                                       // which dev the irq reg belong to
    struct udrv_uio_info* uioinfo;                                                   // 2.1 dev->uioinfo
    unsigned int          missed_nr_interrupts;                                      // ?
    unsigned int          nr_interrupts;                                             // ?
};
```

```blurtext
/* udrv/utils/notification.h */
struct udrv_notification {                                                           // udrv notification obj def
    struct list   link;
    void (*notification_fn)(void* notify_data, void* user_data);
    void*         user_data;
    int           notify_state;
    unsigned long thread_id;
};
```

<p style="margin-bottom: 20px;"></p>

3 init function invoked when load the uio interrupt driver into device driver section:

```blurtext
/* udrv/drivers/driver_uio_interrupt.c */
static int uio_intr_init(struct driver* drv){
    drv->bus = udrv_bus_find_bus(UDRV_INTRDEV_BUS);                                  // set uio irq drv's bus
    return 0;
}
```

<p style="margin-bottom: 20px;"></p>

4 probe function invoked when add new uio irq dev to uio interrupt driver's bus:

```blurtext
static int uio_intr_probe(struct device* dev){
    int ret;
    int fd;
    int monitor_type;
    struct uio_intr_devdata* devdata;                                                // init devdata
    struct invoke_data* invokedata;
    void* handle = NULL;
    devdata = udrv_mem_zalloc(sizeof(*devdata));                                     // alloc mem for devdata
    if(!devdata){
        dev_error(dev, "can not get memory for device data\n");
        return -ENOMEM;
    }
    udrv_dev_set_devdata(dev, devdata);                                              // set dev->devdata = devdata
    if(udrv_ll_dev_interrupt_regs_has_configs(dev)){                                 // 4.1 if dev has "reg" field
        ret = udrv_ll_dev_interrupt_regs_init(dev, &(devdata->regs_context));        // 4.2 init dev's reg ctx
        if(ret < 0){
            return ret;
        }
    }
    devdata->uioname = (char*) udrv_devnode_get_string(dev->devnode, "uioname");     // set devdata->uioname
    devdata->auto_enable = 1;                                                        // set devdata->auto_enable
    if(udrv_ll_dev_has_property(dev, "auto-enable")){                                // override default with custom
        dev_info(dev, "found a valid 'auto-enable' config\n");
        if(!udrv_ll_dev_property_is_true(dev, "auto-enable")){
            devdata->auto_enable = 0;
        }
    }
    dev_info(dev, "interrupt auto-enable: %d\n", devdata->auto_enable);
    DECLARE_INTERRUPT_CONTROLLER_ARGS(intr_ctrl_args);                               // 4.3 setup parent contrller dev
    intr_ctrl_args.operation = INTR_CTRL_OPT_RD_NR_INTERRUPTS_MONITOR_STATE;
    ret = udrv_ll_dev_read(dev->parent, &intr_ctrl_args, &monitor_type, sizeof(int));// get cntl dev monitor type
    if(ret < 0){
        dev_error(dev, "read interrupt controller monitor type failed, error: %d\n", ret);
        return ret;
    }
    devdata->monitor_type = monitor_type;                                            // set irq dev monitor type
    devdata->uioinfo = udrv_plt_uio_open(devdata->uioname);                          // 4.4 try open irq dev udrv uio fd
    if(!devdata->uioinfo){
        dev_error(dev, "open uio file: %s failed\n", devdata->uioname);
        udrv_mem_free(devdata);
        return -EINVAL;
    }
    if(devdata->monitor_type == INTR_DEV_INIT_THREAD_MONITOR){                       // if init the irq reg monitor
        invokedata = udrv_mem_zalloc(sizeof(*invokedata));                           // alloc mem for invoke data
        if(!invokedata){
            return -ENOMEM;
        }
        invokedata->dev = dev;
        fd = udrv_plt_uio_get_fd(devdata->uioinfo);                                  // 4.4 (3) get uio info fd
        ret = update_interrupt_monitor_state(dev, fd, 1, invokedata);                // 4.5 regist fd event to cntl dev
        if(ret < 0){
            return ret;
        }
    }
    else{                                                                            // if other monitor type ????
        ret = udrv_kvmap_create(&devdata->uioinfo_kvmap, key_compare_fn, free_value_fn);  // 4.6 setup uioinfo kvmap
        if(ret < 0){
            dev_error(dev, "create interrupt device fd uioinfo kvmap instance failed, error: %d\n", ret);
            return ret;
        }
        ret = udrv_init_notification_chain(&(devdata->sub_intr_notifications));      // 4.7 init notify chain node
        ret = udrv_subscribe_notification(&(dev->interrupt_notifications),           // 4.8 subscribe dev irq state
                                          current_thread_interrupt_cb,               // 4.9 regist irq notify cb
                                          dev,                                       // dev to subscribe
                                          &handle);
        if(ret < 0){
            dev_error(dev, "init current thread interrupt callback failed, error: %d\n", ret);
            return ret;
        }
    }
    udrv_dev_set_capability(dev, UDRV_DEVICE_CAP_INTERRUPT);                         // 4.10 set dev's cap
    return 0;
}

/* udrv/base/device.h */
enum device_capability {
    UDRV_DEVICE_CAP_INTERRUPT     = 0x00000001,                                      // dev supports irq
    UDRV_DEVICE_CAP_IOWIDTH_08BIT = 0x00000002,                                      // dev supports 8 bit data access
    UDRV_DEVICE_CAP_IOWIDTH_16BIT = 0x00000004,                                      // dev supports 16 bit data access
    UDRV_DEVICE_CAP_IOWIDTH_32BIT = 0x00000010,                                      // dev supports 32 bit data access
    UDRV_DEVICE_CAP_IOWIDTH_64BIT = 0x00000020,                                      // dev supports 64 bit data access
};
```

4.1 judge if dev->devdata has "reg" config field.

```blurtext
/* udrv/ll_services/udrv_ll_interrupt_service.c */
int udrv_ll_dev_interrupt_regs_has_configs(struct device* dev){
    if(!dev || !dev->devnode){
        return 0;
    }
    return udrv_devnode_get_object(dev->devnode, "regs") != NULL;
}
```

4.2 init dev riq reg context: reg's owner dev, regfields...

```blurtext
/* udrv/ll_services/udrv_ll_interrupt_service.c: */
int udrv_ll_dev_interrupt_regs_init(struct device* dev, struct interrupt_regs_context** context){
    int ret;
    struct interrupt_regs_context* this_context;                                     // init the reg ctx
    if(!dev || !context){
        pr_error("invalid arguments, func: %s, line: %d, dev: %p, interrupt_regs_context: %p\n",
                 __func__, __LINE__, dev, context);
        return -EINVAL;
    }
    this_context = udrv_mem_zalloc(sizeof(*this_context));                           // alloc mem for reg ctx
    if(!this_context){
        dev_error(dev, "can not alloc memory for creating 'interrupt_regs_context'\n");
        return -ENOMEM;
    }
    this_context->dev = dev;                                                         // set ctx->dev = cur dev
    list_init(&(this_context->regfields));                                           // init d-linked list node
    ret = udrv_ll_dev_collect_regfield(dev, &(this_context->regfields));             // 4.2.1 collect reg & fill in ctx
    if(ret < 0){
        dev_error(dev, "collect interrupt regs fail, error: %d\n", ret);
        udrv_ll_dev_interrupt_regs_deinit(this_context);
        return ret;
    }
    *context = this_context;
    return 0;
}

/* 4.2.1 udrv/ll_services/udrv_ll_field_service.c: */
int udrv_ll_dev_collect_regfield(struct device* dev, struct list* regfields_out){    // collect reg field lst
    if(!dev || !regfields_out){
        return -EINVAL;
    }
    return udrv_devnode_foreach_regfield(dev->devnode, create_regfield_helper, regfields_out);  // -> 1/2/3/4/5/6
}

/* 1) udrv/base/devnode.c: */
int udrv_devnode_foreach_regfield(devnode_t* devnode,
                                  int (*handle_fn)(const char* name, const char* value,
                                                   const char* option, const char* private_data, void* data),
                                  void* data){
    int         ret;
    const char* value;
    const char* option;
    const char* private_data;
    const char* regfield_name;
    devnode_t*  regfield_nodes;
    devnode_t*  regfield_node;
    regfield_nodes = udrv_devnode_get_object(devnode, "regs");                       // get the reg node lst of dev
    if(!regfield_nodes){
        return -ENODEV;
    }
    devnode_object_foreach(regfield_nodes, regfield_name, regfield_node){            // for each reg node with regname
        option = "";
        if(udrv_devnode_get_object(regfield_node, "option")){                        // node scan with option
            ret = udrv_devnode_scan(regfield_node, "{s:s,s:s,s:s}",
                                    "reg", &value, "option", &option, "privatedata", &private_data);
        }
        else{                                                                        // node scan without option
            ret = udrv_devnode_scan(regfield_node, "{s:s,s:s}", "reg", &value, "privatedata", &private_data); // 2)
        }
        if(ret != 0){
            return ret;
        }
        ret = handle_fn(regfield_name, value, option, private_data, data);           // handle fn for post process
        if(ret){
            return ret;
        }
    }
    return 0;
}

/* 2) udrv/base/devnode.c: */
int udrv_devnode_scan(devnode_t* node, const char* fmt, ...){
    va_list args;
    int ret = 0;
    va_start(args, fmt);
    ret = udrv_devnode_scanv(node, fmt, args);                                       // 3)
    va_end(args);
    return ret;
}

/* 3) udrv/base/devnode.c: */
int udrv_devnode_scanv(devnode_t* node, const char* fmt, va_list args){
    json_error_t error;
    int ret = 0;
    ret = json_vunpack_ex(node, &error, 0, fmt, args);
    if(ret < 0){
        pr_error("Failed unpack json node: %s, source: %s, line: %d, column: %d\n",
                  error.text, error.source, error.line, error.column);
        ret = -EINVAL;
    }
    return ret;
}

/* 4) udrv/basic/field.h */
struct regfield {                                                                    // reg field struct def
    char*       name;
    struct list link;
    char*       dev_name;
    char*       dev_fieldname;
    char*       option;
    int         private_data;
};

/* 5) udrv/ll_services/udrv_ll_field_service.c: handler func -> buildup reg field obj */
static int create_regfield_helper(const char* name,                                  // reg field name
                                  const char* value,
                                  const char* option,
                                  const char* private_data,                          // priv date
                                  void* data){
    struct list*     regfields = (struct list*) data;                                // 4)
    struct regfield* field;
    char*            dev_name;
    char*            dev_fieldname;
    dev_fieldname = udrv_str_str((char*) value, ".");
    if(!dev_fieldname || *(dev_fieldname + 1) == '\0'){
        return -EINVAL;
    }
    dev_name = udrv_str_n_dup((char*)value, dev_fieldname - (char*)value);
    dev_fieldname += 1;
    field = udrv_field_create_regfield((char*)name, dev_name, dev_fieldname, (char*)option, (char*)private_data); // 6
    if(field){
        list_append(regfields, &(field->link));
    }
    udrv_str_free(dev_name);
    return 0;
}

/* 6) udrv/ll_services/udrv_ll_field_service.c */
struct regfield* udrv_field_create_regfield(char* name,                              // core func to buildup regfield
                                            char* dev_name,
                                            char* dev_fieldname,
                                            char* option,
                                            char* private_data){
    struct regfield* f = udrv_mem_zalloc(sizeof(*f));
    if(f){
        f->name = udrv_str_dup(name);
        f->dev_name = udrv_str_dup(dev_name);
        f->dev_fieldname = udrv_str_dup(dev_fieldname);
        f->option = udrv_str_dup(option);
        list_init(&(f->link));
        if(udrv_str_to_int(private_data, &f->private_data)){
            udrv_field_free_regfield(f);
            f = NULL;
        }
    }
    return f;
}
```

4.3 irq controller args, passed by irq dev to register event in controller epoll for monitor.

```blurtext
/* udrv/ll_interfaces/udrv_ll_driver_arguments.h */
struct interrupt_controller_args {                                                   // init irq cntrl args obj
    int      is_register;                                                            // register or unregister
    int      fd;                                                                     // fd to monitor
    int      operation;                                                              // operaion type
    void (*interrupt_cb)(void* data);                                                // cb when irq happen (fd)
    void*    data;                                                                   // cb args
};
#define DECLARE_INTERRUPT_CONTROLLER_ARGS(args) struct interrupt_controller_args args = {0, -1, 0, NULL, 0}
```

4.4 platform interfaces: devdata->uioinfo related set & get api definition.

```blurtext
/* udrv/platform/plt_adapter.c */
static struct plt_interfaces default_plt_interfaces = {                              // uio related interfaces
    ...
    .udrv_plt_uio_open_fn = udrv_plt_default_uio_open,                               // 1)
    .udrv_plt_uio_close_fn = udrv_plt_default_uio_close,                             // 2)
    .udrv_plt_uio_get_fd_fn = udrv_plt_default_uio_get_fd,                           // 3) udrv/tools/target_shell.c
    .udrv_plt_uio_enable_irq_fn = udrv_plt_default_uio_enable_irq,
    .udrv_plt_uio_disable_irq_fn = udrv_plt_default_uio_disable_irq,
    .udrv_plt_uio_get_mem_map_fn = udrv_plt_default_uio_get_mem_map,

    .udrv_plt_uio_generic_read_reg_ex_fn = udrv_plt_default_uio_generic_read_reg,
    .udrv_plt_uio_generic_write_reg_ex_fn = udrv_plt_default_uio_generic_write_reg,
    ...
};

/* 1) udrv/platform/plt_adapter.c: init & get dev->devdata uioinfo */
static struct udrv_uio_info* udrv_plt_default_uio_open(char* uioname){
    struct udrv_uio_info* uioinfo;                                                   // init uioinfo struct
    int ret;
    if(!uioname){
        return NULL;
    }
    uioinfo = udrv_mem_zalloc(sizeof(*uioinfo));                                     // alloc mem for uioinfo
    if(!uioinfo){
        return NULL;
    }
    uioinfo->uioname = udrv_str_dup(uioname);                                        // set uio name
    uioinfo->size = UIO_MAPPED_MEM_SIZE;                                             // set uio mmap mem size
    uioinfo->vaddr = udrv_mem_zalloc(UIO_MAPPED_MEM_SIZE);                           // set mmap vaddr
    if(!uioinfo->vaddr){
        goto fail;
    }
    fill_uio_device_memory(uioinfo);                                                 // a) fill the uioinfo mem
    if(access(uioinfo->uioname, F_OK)){                                              // if uio file existed (uio fw?)
        ret = mkfifo(uioinfo->uioname, 0755);                                        // init named fifo pipe (buffer)
        if(ret){
            goto fail;
        }
    }
    uioinfo->fd = udrv_file_open(uioinfo->uioname, O_RDONLY | O_NONBLOCK);           // b) set the uio file fd
    if(uioinfo->fd < 0){
        goto fail;
    }
    return uioinfo;
fail:                                                                                // cleanups
    udrv_plt_default_uio_close(uioinfo);
    return NULL;
}

/* a) udrv/platform/plt_adapter.c */
static void fill_uio_device_memory(struct udrv_uio_info* uioinfo){
    int i;
    unsigned char* p = (unsigned char*)uioinfo->vaddr;                               // get uio mem start addr
    for(i = 0; i < uioinfo->size; i++){                                              // fill seq num in uio mem
        p[i] = (unsigned char)i;
    }
}

/* b) udrv/utils/utils.c */
int udrv_file_open(char* pathname, int flags){
    int fd;
    while((fd = open(pathname, flags)) < 0){                                         // try open until success
        if(errno != EINTR){                                                          // not interrupted by sig, no retry
            return -errno;
        }
    }
    return fd;
}

/* 2) udrv/platform/plt_adapter.c */
static int udrv_plt_default_uio_close(struct udrv_uio_info* info){
    if(!info{
        return -EINVAL;
    }
    if(info->vaddr){
        udrv_mem_free(info->vaddr);
    }
    if(info->fd > 0){
        udrv_file_close(info->fd);
    }
    udrv_mem_free(info);
    return 0;
}

/* 3) udrv/platform/plt_adapter.c */
int udrv_plt_uio_get_fd(struct udrv_uio_info* info){                                 // wrapper: ret uio info fd
    if(user_interfaces_ptr && user_interfaces_ptr->udrv_plt_uio_get_fd_fn){
        return user_interfaces_ptr->udrv_plt_uio_get_fd_fn(info);
    }
    return default_interfaces_ptr->udrv_plt_uio_get_fd_fn(info);
}

/* 3) udrv/platform/plt_adapter.c */
static int udrv_plt_default_uio_get_fd(struct udrv_uio_info* info){                  // core: ret udrv_uio_info fd
    return info ? info->fd : -EINVAL;
}
```

4.5 register event in controller dev drv's io monitor model: epoll.

```blurtext
/* udrv/drivers/driver_uio_interrupt.c */
static int update_interrupt_monitor_state(struct device* dev, int fd, int is_register, struct invoke_data* data){
    int ret;
    DECLARE_INTERRUPT_CONTROLLER_ARGS(intrcargs);                                    // 4.3 setup nit irq cntl dev args
    intrcargs.is_register = is_register;                                             // register monitor in cntl dev
    intrcargs.fd = fd;                                                               // set fd to monitor
    intrcargs.interrupt_cb = udrv_dev_uio_interrupt_cb;                              // 1) cb when dev irq triggered
    intrcargs.data = data;                                                           // cb args
    ret = udrv_ll_dev_write(dev->parent, &intrcargs, NULL, 0);                       // 2) add irq event to cntl dev
    return ret;
}

/* udrv/ll_interfaces/udrv_ll_driver_arguments.h */
#define interrupt_device_argument_members  int operation;
struct interrupt_device_args {
    interrupt_device_argument_members
    int fd;
};
#define DECLARE_INTERRUPT_DEVICE_ARGS(args) struct interrupt_device_args args = \
    {INTR_DEV_OPT_RD_NR_INTERRUPTS_FROM_HISTORY, 0}

/* 1) udrv/drivers/driver_uio_interrupt.c */
static void udrv_dev_uio_interrupt_cb(void* args){
    struct invoke_data* invokedata = (struct invoke_data*)args;                      // init cb args: invoke data
    struct device* dev = invokedata->dev;                                            // get the dev the irq from
    struct udrv_uio_info* uioinfo = invokedata->uioinfo;
    struct uio_intr_devdata* devdata;                                                // init devdata
    DECLARE_INTERRUPT_DEVICE_ARGS(intr_args);                                        // init irq dev args
    unsigned int nr_interrupts = 0;
    unsigned int missed_nr_interrupts = 0;
    int ret;
    int invoke_fd;
    if(!dev){
        pr_error("invalid arguments, func: %s, line: %d, dev: %p\n", __func__, __LINE__, dev);
        return;
    }
    devdata = udrv_dev_get_devdata(dev);                                             // get dev->devdata
    if(!devdata){
        dev_error(dev, "not found the device data\n");
        return;
    }
    if(devdata->monitor_type == INTR_DEV_INIT_THREAD_MONITOR){                       // if use init thd for monitor
        invoke_fd = udrv_plt_uio_get_fd(devdata->uioinfo);                           // get uio fd from devdata
    }
    else{
        invoke_fd = udrv_plt_uio_get_fd(uioinfo);                                    // get uio fd from uioinfo
    }
    dev_debug(dev, "read interrupts from uio: %s, fd: %d\n", devdata->uioname, invoke_fd);
    intr_args.operation = INTR_DEV_OPT_RD_NR_INTERRUPTS_FROM_HARDWARE;
    intr_args.fd = invoke_fd;                                                        // set epoll monitor fd as uio fd
    ret = udrv_ll_dev_read(dev, &intr_args, &nr_interrupts, sizeof(unsigned int));   // 3) dev->read to get statistics
    if(ret < 0){
        dev_error(dev, "read uio interrupts number failed, error: %d\n", ret);
        return;
    }
    dev_info(dev, "uio interrupts fd: %d, uio interrupts number: %u\n", invoke_fd, nr_interrupts);
    missed_nr_interrupts = nr_interrupts - invokedata->nr_interrupts - 1;            // compute missed irqs
    if(missed_nr_interrupts > 0){
        invokedata->missed_nr_interrupts += missed_nr_interrupts;
        dev_info(dev, "uio interrupts fd: %d, uio missed total interrupts number : %u\n",
                 invoke_fd, invokedata->missed_nr_interrupts);
    }
    invokedata->nr_interrupts = nr_interrupts;
    if(invokedata->nr_interrupts > devdata->nr_interrupts){
        devdata->nr_interrupts = invokedata->nr_interrupts;
    }
    if(devdata->monitor_type == INTR_DEV_INIT_THREAD_MONITOR){
        devdata->missed_nr_interrupts = invokedata->missed_nr_interrupts;
    }
    udrv_ll_dev_invoke_interrupt_notifications(dev);
    if((devdata->auto_enable) && (!udrv_ll_dev_is_interrupt_notification_empty(dev))){
        ret = udrv_plt_uio_enable_irq(devdata->uioinfo);
        if(ret){
            dev_warn(dev, "auto enable interrupt failed, error: %d\n", ret);
        }
        else{
            dev_debug(dev, "auto enable interrupt success\n");
        }
    }
}

/* 2) udrv/ll_services/udrv_ll_general_io_service.c */
int udrv_ll_dev_write(struct device* dev, void* args, void* data, int size){         // write to dev devdata cfg args
    if(dev && dev->drv && dev->drv->opt && dev->drv->opt->write){
        return dev->drv->opt->write(dev, args, data, size);
    }
    pr_error("No implementation for the WRITE operation or no driver serves this device\n");
    return -EINVAL;
}

/* 3) udrv/ll_services/udrv_ll_general_io_service.c */
int udrv_ll_dev_read(struct device* dev, void* args, void* data, int size){
    if(dev && dev->drv && dev->drv->opt && dev->drv->opt->read){
        return dev->drv->opt->read(dev, args, data, size);
    }
    pr_error("No implementation for the READ operation or no driver serves this device\n");
    return -EINVAL;
}
```

4.6 udrv kvmap data structure creation helper function (suppport customized compare & free fn):

```blurtext
/* udrv/utils/kvmap.c */
int udrv_kvmap_create(struct udrv_kvmap** kvmap,                                     // create & ret udrv kvmap
                      int (*key_compare_fn)(const void* key1, const void* key2),
                      void (*free_value_fn)(void* value)){
    struct udrv_kvmap* kv;                                                           // init kvmap
    if(!kvmap || !key_compare_fn){
        pr_error("invalid arguments, func: %s, line: %d, kvmap: %p, key_compare_fn: %p\n",
                 __func__, __LINE__, kvmap, key_compare_fn);
        return -EINVAL;
    }
    kv = udrv_mem_zalloc(sizeof(*kv));                                               // alloc mem for kvmap
    if(!kv){
        pr_error("can not alloc memory for kvmap instance\n");
        return -ENOMEM;
    }
    list_init(&(kv->elements));                                                      // create lst node by kv ele
    kv->key_compare_fn = key_compare_fn;                                             // 1) set key cmp func
    kv->free_value_fn = free_value_fn;                                               // 2) set ?
    *kvmap = kv;
    return 0;
}

/* 1) udrv/drivers/driver_uio_interrupt.c */
static int key_compare_fn(const void* key1, const void* key2){                       // judge if 2 notify the same
    return (struct udrv_notification*)key1 == (struct udrv_notification*)key2;
}

/* 2) udrv/drivers/driver_uio_interrupt.c */
static void free_value_fn(void* value){
    struct udrv_uio_info* uioinfo = (struct udrv_uio_info*)value;
    udrv_plt_uio_close(uioinfo);
}
```

4.7 udrv init device level notification chain function:

```blurtext
/* udrv/utils/notification.c */
int udrv_init_notification_chain(struct udrv_notification_chain* chain){             // init dev level notification lst
    if(!chain){
        return -EINVAL;
    }
    list_init(&(chain->notifications));                                              // init notification lst node
    return 0;
}
```

4.8 use extra thread to subscribe udrv notification in udrv by register udrv_notification ctx.

```blurtext
/* udrv/utils/notification.c */
int udrv_subscribe_notification(struct udrv_notification_chain* chain,
                                void (*notification_fn)(void* notify_data, void* user_data),
                                void* user_data,
                                void** handle){
    return udrv_subscribe_notification_on_thread(chain,
                                                 NON_AFFINITY_THREAD_ID,
                                                 notification_fn,
                                                 user_data,
                                                 handle);
}

/* udrv/utils/notification.c */
static int udrv_subscribe_notification_on_thread(struct udrv_notification_chain* chain,
                                                 unsigned long thread_id,
                                                 void (*notification_fn)(void* notify_data, void* user_data),
                                                 void* user_data,
                                                 void** handle){
    struct udrv_notification* notification;
    if(!chain || !notification_fn || !handle){
        return -EINVAL;
    }
    notification = udrv_mem_zalloc(sizeof(*notification));
    if(!notification){
        return -ENOMEM;
    }
    notification->notification_fn = notification_fn;
    notification->user_data = user_data;
    notification->notify_state = UDRV_NOTIFY_STATE_INIT;
    notification->thread_id = thread_id;
    list_append(&(chain->notifications), &(notification->link));
    *handle = notification;
    return 0;
}
```

4.9 use 'current thread' to notify the dev by calling cb fn registered in udrv_notification (from chain).

```blurtext
/* udrv/drivers/driver_uio_interrupt.c */
static void current_thread_interrupt_cb(void* notify_data, void* device){
    unsigned long current_thread_id = 0;
    struct device* dev = (struct device*)device;
    struct uio_intr_devdata* devdata = (struct uio_intr_devdata*)dev->devdata;
    struct udrv_notification_chain* chain = &devdata->sub_intr_notifications;
    struct udrv_notification* n = NULL;
    struct udrv_notification* t = NULL;
    if(!chain){
        dev_error(dev, "interrupt dev data don't have sub chain , func: %s, line: %d\n", __func__, __LINE__);
        return;
    }
    list_for_each_entry(n, &(chain->notifications), link){
        n->notify_state = UDRV_NOTIFY_STATE_INIT;
    }
    udrv_plt_thread_get_current(&current_thread_id);
    list_for_each_entry_safe_more(n, t, &(chain->notifications), link){
        if(current_thread_id == n->thread_id){
            if(n->notify_state == UDRV_NOTIFY_STATE_INIT){
                n->notify_state = UDRV_NOTIFY_STATE_DONE;
                n->notification_fn(notify_data, n->user_data);
            }
        }
    }
}
```

4.10 udrv set dev's capability helper function.

```blurtext
/* udrv/base/device.c */
int udrv_dev_set_capability(struct device* dev, int capability){
    if(!dev){
        return -EINVAL;
    }
    udrv_dev_lock(dev);                                                              // 1 lock to protect mult-dev probe
    dev->capabilities |= capability;                                                 // add cap to dev instance
    udrv_dev_unlock(dev);                                                            // 2 unlock
    return 0;
}

/* 1 udrv/base/device.c */
int udrv_dev_lock(struct device* dev){                                               // lock dev
    int ret;
    if(!dev){
        return -EINVAL;
    }
    ret = udrv_plt_lock(dev->lock);
    if(ret == 0){
        dev->locked = 1;
    }
    return ret;
}

/* 1 udrv/platform/plt_adapter.c */
int udrv_plt_lock(struct udrv_lock* lock){
    if(user_interfaces_ptr && user_interfaces_ptr->udrv_plt_lock_fn){
        return user_interfaces_ptr->udrv_plt_lock_fn(lock);
    }
    return default_interfaces_ptr->udrv_plt_lock_fn(lock);                           // call plt api lock
}

/* 1 udrv/platform/plt_adapter.c */
static int udrv_plt_default_lock(struct udrv_lock* lock){                            // plt api lock
    return lock ? pthread_mutex_lock(&lock->lock) : -EINVAL;
}

/* 2 udrv/base/device.c */
int udrv_dev_unlock(struct device* dev){                                             // unlock dev
    int ret;
    if(!dev){
        return -EINVAL;
    }
    dev->locked = 0;                                                                 // update dev state as unlock
    ret = udrv_plt_unlock(dev->lock);                                                // call plt api unlock
    return ret;
}

/* 2 udrv/platform/plt_adapter.c */
int udrv_plt_unlock(struct udrv_lock* lock){
    if(user_interfaces_ptr && user_interfaces_ptr->udrv_plt_unlock_fn){
        return user_interfaces_ptr->udrv_plt_unlock_fn(lock);
    }
    return default_interfaces_ptr->udrv_plt_unlock_fn(lock);
}

/* 2 udrv/platform/plt_adapter.c */
static int udrv_plt_default_unlock(struct udrv_lock* lock){                          // plt api lock
    return lock ? pthread_mutex_unlock(&lock->lock) : -EINVAL;
}
```

<p style="margin-bottom: 20px;"></p>

5 uio irq device driver supported operation registration.

```blurtext
static struct drv_operation uio_intr_opts = {
    .read = uio_intr_read,
    .write = uio_intr_write,
};
```

```blurtext
static int uio_intr_read(struct device* dev, void* args, void* data, int size){
    struct uio_intr_devdata*      devdata;
    struct interrupt_device_args* intr_args;
    if(!dev || !args || !data || size < sizeof(int)){
        pr_error("invalid arguments, func: %s, line: %d, dev: %p, args: %p, data: %p, size: %d\n",
                 __func__, __LINE__, dev, args, data, size);
        return -EINVAL;
    }
    devdata = udrv_dev_get_devdata(dev);
    if(!devdata){
        dev_error(dev, "not found the device data\n");
        return -EINVAL;
    }
    intr_args = (struct interrupt_device_args*)args;
    if(intr_args->operation == INTR_DEV_OPT_RD_NR_INTERRUPTS_FROM_HISTORY){
        *(unsigned int*)data = devdata->nr_interrupts;                               // ret num of irqs
        return 0;
    }
    if(intr_args->operation == INTR_DEV_OPT_RD_NR_MISSED_INTERRUPTS_FROM_HISTORY){
        *(unsigned int*)data = devdata->missed_nr_interrupts;                        // ret num of missed irqs
        return 0;
    }
    if(intr_args->operation == INTR_DEV_OPT_RD_NR_INTERRUPTS_MONITOR_TYPE){
        *(int*)data = devdata->monitor_type;                                         // ret dev's monitor type
        return 0;
    }
    if(intr_args->operation == INTR_DEV_OPT_RD_NR_INTERRUPTS_FROM_HARDWARE){         //
        return udrv_file_read(intr_args->fd, data, size);                            // read size bytes to data from fd
    }
    dev_error(dev, "invalid read operation: %d, error: %d\n", intr_args->operation, -EINVAL);
    return -EINVAL;
}
```

```blurtext
static int uio_intr_write(struct device* dev, void* args, void* data, int size){
    struct uio_intr_devdata*    devdata;
    struct udrv_uio_info*       uioinfo;
    struct interrupt_ctrl_args* ctrl_args = (struct interrupt_ctrl_args*) args;
    struct invoke_data* invokedata;
    struct subscription_internal_handle* internal_handle;
    int    ret;
    int    fd;
    int    has_notification = 0;
    if(!dev || !args){
        pr_error("invalid arguments, func: %s, line: %d, dev: %p, args: %p\n",
                 __func__, __LINE__, dev, args);
        return -EINVAL;
    }
    devdata = udrv_dev_get_devdata(dev);
    if(!devdata){
        dev_error(dev, "not found the device data\n");
        return -EINVAL;
    }
    if(ctrl_args->interrupt_operation == UNSUBSCRIBE_INTERRUPT_WITH_CURRENT_THREAD){
        if(ctrl_args->handle == NULL){
            dev_error(dev, "get interrupt handle failed\n");
            return -EINVAL;
        }
        ret = udrv_kvmap_get(devdata->uioinfo_kvmap, ctrl_args->handle, (void**)&uioinfo);
        if(ret < 0){
            dev_error(dev, "get uioinfo failed, key handle:%p \n", ctrl_args->handle);
            return ret;
        }
        fd = udrv_plt_uio_get_fd(uioinfo);
        internal_handle = (struct subscription_internal_handle*) ctrl_args->handle;
        ret = udrv_unsubscribe_notification(&(devdata->sub_intr_notifications), internal_handle->handle);
        if(ret < 0){
            dev_error(dev, "unsubscribe %p handle failed, error: %d\n", ctrl_args->handle, ret);
        }
        udrv_mem_free(internal_handle->data);
        udrv_mem_free(internal_handle);
        ret = update_interrupt_monitor_state(dev, fd, 0, NULL);
        if(ret < 0){
            dev_error(dev, "unregister %p handle with fd: %d to interrupt controller failed, error: %d\n",
                      ctrl_args->handle, fd, ret);
        }
        has_notification = udrv_is_chain_empty(&(devdata->sub_intr_notifications));
        if((ret == 0) && !has_notification){
            ret = udrv_ll_dev_disable_interrupt(dev);
        }
        ret = udrv_kvmap_del(devdata->uioinfo_kvmap, ctrl_args->handle);
        if(ret < 0){
            dev_error(dev, "delete handle: %p uioinfo failed, error: %d\n", ctrl_args->handle, ret);
        }
        return ret;
    }
    if(ctrl_args->interrupt_operation == SUBSCRIBE_INTERRUPT_WITH_CURRENT_THREAD &&
        devdata->monitor_type == INTR_DEV_CURRENT_THREAD_MONITOR){
        if(!ctrl_args->notification_fn){
            dev_error(dev, "inerrupt notification callback is NULL\n");
            return -EINVAL;
        }
        uioinfo = udrv_plt_uio_open(devdata->uioname);
        if(!uioinfo){
            dev_error(dev, "open uio file: %s failed\n", devdata->uioname);
            return -EINVAL;
        }
        fd = udrv_plt_uio_get_fd(uioinfo);
        invokedata = udrv_mem_zalloc(sizeof(*invokedata));
        if(!invokedata){
            return -ENOMEM;
        }
        invokedata->dev = dev;
        invokedata->uioinfo = uioinfo;
        internal_handle = udrv_mem_zalloc(sizeof(*internal_handle));
        if(!internal_handle){
            udrv_mem_free(invokedata);
            return -ENOMEM;
        }
        internal_handle->data = invokedata;
        ret = udrv_subscribe_notification_with_thread(
                  &(devdata->sub_intr_notifications),
                  ctrl_args->notification_fn,
                  ctrl_args->user_data,
                  &(internal_handle->handle));
        ctrl_args->handle = internal_handle;
        if(ret < 0){
            dev_error(dev, "subscribe interrupt with thread failed, error: %d\n", ret);
            udrv_plt_uio_close(uioinfo);
            return ret;
        }
        ret = update_interrupt_monitor_state(dev, fd, 1, invokedata);
        if(ret < 0){
            dev_error(dev, "register fd: %d to interrupt controller failed, error: %d\n", fd, ret);
            udrv_plt_uio_close(uioinfo);
            udrv_unsubscribe_notification(&(devdata->sub_intr_notifications), ctrl_args->handle);
            return ret;
        }
        udrv_kvmap_set(devdata->uioinfo_kvmap, ctrl_args->handle, uioinfo);
        return ret;
    }
    if(udrv_ll_dev_interrupt_is_storm_operation(ctrl_args->interrupt_operation)){
        if(devdata->regs_context){
            return udrv_ll_dev_interrupt_storm_operation(dev,
                                                         devdata->regs_context,
                                                         ctrl_args->interrupt_operation,
                                                         &(ctrl_args->interrupt_state));
        }
        // If storm status is obtained, since there is no regs_context,
        // we return 0, which means there was never a storm
        ctrl_args->interrupt_state = 0;
        return 0;
    }
    if(ctrl_args->interrupt_operation == UDRV_INTERRUPT_ENABLE){
        ret = udrv_plt_uio_enable_irq(devdata->uioinfo);
        if(ret == 0){
            dev_info(dev, "device interrupt is enabled\n");
        }
        return ret;
    }
    if(ctrl_args->interrupt_operation == UDRV_INTERRUPT_DISABLE){
        ret = udrv_plt_uio_disable_irq(devdata->uioinfo);
        if(ret == 0){
            dev_info(dev, "device interrupt is disabled\n");
        }
        return ret;
    }
    return 0;
}
```

<hr>

### # udrv.json example for uio interrupt device driver

```blurtext
{
    "configs" : {                                                                    // basic udrv dlib program config
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

    "intrc": {
        "compatible": "udrv interrupt controller",                                   // level I irq controller driver

        "intr_ntio_presence": {                                                      // ntio presence dev ?
            "compatible": "udrv uio interrupt device",                               // level II device driver
            "uioname": "uio_pipe_ntio_presence",                                     // ?
            "auto-enable": false,                                                    // ?
            "regs": {                                                                // reg config
                "storm": { "reg": "intr_storm.storm", "privatedata": "0x1" }         // if intr_storm.storm == 0x1
            }                                                                        // then ntio is present ?
        },

        "intr_sfp0_presence": {
            "compatible": "udrv uio interrupt device",
            "uioname": "uio_pipe_sfp0_presence"
        },

        "intr_sfp0_rx_los": {
            "compatible": "udrv uio interrupt device",
            "uioname": "uio_pipe_sfp0_rx_los"
        },

        "intr_sfp0_tx_fault": {
            "compatible": "udrv uio interrupt device",
            "uioname": "uio_pipe_sfp0_tx_fault"
        },

        "intr_alarm0": {
            "compatible": "udrv uio interrupt device",
            "uioname": "uio_pipe_alarm0"
        },

        "intr_alarm1": {
            "compatible": "udrv inotify device",
            "pathname": "vroot/inotify/inotify_alarm1",
            "events": ["IN_CLOSE_WRITE"],
            "regs": {
                "storm": { "reg": "intr_storm.storm", "privatedata": "0x1" }
            }
        },

        "intr_inotify_test": {
            "compatible": "udrv inotify device",
            "pathname": "vroot/inotify/inotify_test",
            "auto-enable": false
        }
    },
}

static int udrv_subscribe_notification_with_thread(
    struct udrv_notification_chain* chain,
    void (*notification_fn)(void* notify_data, void* user_data),
    void*  user_data,
    void** handle){
    struct udrv_notification* notification;
    if(!chain || !notification_fn || !handle){
        return -EINVAL;
    }
    notification = udrv_mem_zalloc(sizeof(*notification));
    if(!notification){
        return -ENOMEM;
    }
    notification->notification_fn = notification_fn;
    notification->user_data = user_data;
    notification->notify_state = UDRV_NOTIFY_STATE_INIT;
    udrv_plt_thread_get_current(&notification->thread_id);
    list_append(&(chain->notifications), &(notification->link));
    *handle = notification;
    return 0;
}

struct subscription_internal_handle {
    struct invoke_data* data;
    void* handle;
};
```

<hr>

### reference
1 access syscall: https://man7.org/linux/man-pages/man2/access.2.html
