---
layout: post
title: "udrv: interrupt controller driver code walk I (linux, udrv)"
author: "melon"
date: 1111-07-23 17:09
categories: "2024"
tags:
  - linux
  - driver
---

this article will focus on the main trunk of the irq controller driver, which is widely used in both target &
emulation userspace platform apps for facilitating hardware access operations.

<hr>

### # code walk of controller driver of interrupt device (from down to top)
1 driver basic struct definition & driver section setup:

```blurtext
/* udrv/drivers/driver_interrupt_controller.c */
static struct driver interrupt_controller_driver = {
    .name = "udrv interrupt controller driver",              // drv name
    .compatible = "udrv interrupt controller",               // for matching of dev <-> drv
    .init = intrc_init,                                      // cb func called right before this drv registered in udrv
    .probe = intrc_probe,                                    // cb func called when add new dev under this drv's bus
    .opt = &interrupt_controller_opts,                       // supported operations
};

/* udrv/drivers/driver_interrupt_controller.c */
udrv_controller_driver(interrupt_controller_driver);         // set irq controller drv into controller driver section
```

<p style="margin-bottom: 20px;"></p>

2 interrupt controller device backend data structure definitions (the devdata):

```blurtext
/* udrv/drivers/driver_interrupt_controller.c */
struct interrupt_controller_devdata {
    int is_custom_monitor_method;                            // whether use customized monitor method
    int controller_monitor_type;                             // monitor type
    struct udrv_interrupt_monitor_context* monitor_context;  // 2.1 monitor context definition
};
```

2.1 udrv irq dev monitor context info data structure definition:

```blurtext
/* udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
struct udrv_interrupt_monitor_context {
    struct device* dev;                                      // irq of which dev to monitor
    int is_custom_monitor_method;                            // flag of the monitor method
    struct custom_context* custom;                           // 2.1.1 custom ctx (if use custom method)
    struct epoll_context* epoll;                             // 2.1.2 epoll ctx (if use epoll)
};
```

2.1.1 & 2.1.2 lowest level context stucture for monitor method: custom & epoll.

```blurtext
/* udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
#define MAX_INTERRUPT_DEVICES              256               // 1 thread notify <= 256 subscriber dev
```

```blurtext
/* 2.1.1 udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
struct custom_context {
    struct custom_event events[MAX_INTERRUPT_DEVICES];       // custom event list (up to 256 dev to notify)
}

/* udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
struct custom_event {                                        // each custom event to trigger <=> 1 monitoring fd
    int    fd;
    void*  handle;
};
```

```blurtext
/* 2.1.2 udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
struct epoll_context {
    struct udrv_epoll_context context;                       // 1) udrv epoll ctx (fd, events)
    struct udrv_thread*       thread;                        // 2) udrv thread
    struct udrv_epoll_event   events[MAX_INTERRUPT_DEVICES]; // 3) udrv epoll event list (up to 256 irq dev to notify)
};

/* 1) udrv/utils/epoll.h */
struct udrv_epoll_context {                                  // each epoll obj <=> multiple event to trigger
    int                 epoll_fd;                            // epoll instance fd
    struct epoll_event* events;                              // events to trigger when reg fd ready
    int                 max_events;
};

/* 2) udrv/platform/plt_adapter.c */
struct udrv_thread {                                         // udrv thread ctx
    pthread_t           thread;                              // pthread obj
    void* (*user_start_routine)(void*);                      // thread job func
    void*               user_args;                           // thread args
};

/* 3) udrv/utils/epoll.h */
struct udrv_epoll_event {                                    // udrv event listening to epoll fd
    int                 fd;                                  // fd for epoll to monitor
    int                 events;                              // epoll event additional info
    void (*event_cb)(void* event);                           // cb when the udrv event triggered
    void*               data[4];                             // epoll user data placeholder
};
```

<p style="margin-bottom: 20px;"></p>

3 init function invoked when load the interrupt controller driver into controller driver section:

```blurtext
/* udrv/drivers/driver_interrupt_controller.c */
static int intrc_init(struct driver* drv){
    drv->bus = udrv_bus_find_bus(UDRV_PLATFROM_BUS);         // set interrupt controller driver's bus as platform bus
    return 0;
}
```

<p style="margin-bottom: 20px;"></p>

4 probe function invoked when add new abstract dev to interrupt controller driver's bus:

```blurtext
/* udrv/drivers/driver_interrupt_controller.c */
static int intrc_probe(struct device* dev){                                     // newly probed irq dev as publisher
    int ret;
    struct bus_type* bus;
    struct interrupt_controller_devdata* devdata;                               // init new-add matched dev->devdata
    char* monitor_method;
    struct udrv_interrupt_monitor_config monitor_config;                        // 4.1 init irq monitor cfg
    devdata = udrv_mem_zalloc(sizeof(*devdata));
    if(!devdata){
        dev_error(dev, "can not get memory for creating device data\n");
        return -ENOMEM;
    }
    udrv_dev_set_devdata(dev, devdata);                                         // set dev->devdata = devdata
    monitor_method = (char*)udrv_ll_dev_property_string(dev, "monitor-method"); // [1] set devdata monitor thd info
    if(!monitor_method || !udrv_str_equals(monitor_method, "custom")){          // if no use custom monitor method
        dev_info(dev, "use epoll thread to monitor interrupt\n");               // use epoll thread to monitor irq
        devdata->is_custom_monitor_method = 0;
        devdata->controller_monitor_type = INTR_DEV_INIT_THREAD_MONITOR;        // set monitor type
    }
    else{                                                                       // use customized irq monitor method
        dev_info(dev, "use custom method to monitor interrupt\n");
        devdata->is_custom_monitor_method = 1;
        if(udrv_ll_dev_property_is_true(dev, "thread-affinity")){               // if set thread affinity
            dev_info(dev, "use caller thread to monitor interrupt\n");          // use caller thread to monitor irq
            devdata->controller_monitor_type = INTR_DEV_CURRENT_THREAD_MONITOR; // set monitor type
        }
        else{                                                                   // no set thread affinity
            dev_info(dev, "use init thread to monitor interrupt\n");            // use init thread to monitor irq
            devdata->controller_monitor_type = INTR_DEV_INIT_THREAD_MONITOR;    // set monitor type
        }
    }
    if(devdata->is_custom_monitor_method == 0){                                 // [2] set monitor config thd prio
        udrv_ll_dev_property_integer(dev, "monitor-thread-priority", &monitor_config.thread_priority);
        if(monitor_config.thread_priority <= 0){
            monitor_config.thread_priority = INTERRUPT_MONITOR_THREAD_DEFAULT_PRIORITY;
        }
        dev_info(dev, "interrupt monitor thread priority: %d\n", monitor_config.thread_priority);
    }
    ret = udrv_ll_dev_interrupt_monitor_init(dev,                               // [3] 4.2 init devdata->monitor ctx
                                             &devdata->monitor_context,
                                             monitor_config,
                                             devdata->is_custom_monitor_method);
    if(ret){
        dev_error(dev, "init interrupt monitor service failed, error: %d\n", ret);
        udrv_mem_free(devdata);
        return ret;
    }
    bus = udrv_bus_find_bus(UDRV_INTRDEV_BUS);                                  // [4] get udrv interrupt bus obj
    ret = udrv_dev_populate_next_level_devices(dev, bus);                       // set next level matched dev's bus
    if(ret){
        dev_error(dev, "populate sub-devices failed, error: %d\n", ret);
        udrv_ll_dev_interrupt_monitor_deinit(devdata->monitor_context);
        udrv_mem_free(devdata);
        return ret;
    }
    return 0;
}
```

4.1 interrupt monitor thread configuration.

```blurtext
/* udrv/ll_services/udrv_ll_interrupt_monitor.h */
struct udrv_interrupt_monitor_config {
    int thread_priority;
};
```

4.2 init irq dev monitor thread context

```blurtext
/* udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
int udrv_ll_dev_interrupt_monitor_init(struct device* dev,                                       // irq dev to monitor
                                       struct udrv_interrupt_monitor_context** monitor_context,  // for mod ctx ptr val
                                       struct udrv_interrupt_monitor_config config,
                                       int is_custom_monitor_method){
    int ret;
    struct udrv_interrupt_monitor_context* this_monitor_context;                         // [1] 2 init irq monitor ctx
    int memory_size;
    if(!dev || !monitor_context){
        pr_error("invalid arguments, func: %s, line: %d, dev: %p, monitor_context %p\n",
                 __func__, __LINE__, dev, monitor_context);
        return -EINVAL;
    }
    memory_size = sizeof(struct udrv_interrupt_monitor_context);                         // [2] set ctx mem size
    if(is_custom_monitor_method){                                                        // add custom ctx size if need
        memory_size += sizeof(struct custom_context);
    }
    else{                                                                                // add epoll ctx size if need
        memory_size += sizeof(struct epoll_context);
    }
    this_monitor_context = udrv_mem_zalloc(memory_size);                                 // zalloc mem
    if(!this_monitor_context){
        dev_error(dev, "can not alloc memory for creating 'monitor_context'\n");
        return -ENOMEM;
    }
    this_monitor_context->is_custom_monitor_method = is_custom_monitor_method;           // [3] set monitor ctx flag
    this_monitor_context->dev = dev;                                                     // [4] set monitor irq dev
    if(is_custom_monitor_method){
        this_monitor_context->custom = (struct custom_context*) &this_monitor_context[1];// set ctx->custom
        ret = udrv_ll_dev_custom_monitor_init(dev, this_monitor_context->custom);        // [5] 4.3 set custom ctx
    }
    else{
        this_monitor_context->epoll = (struct epoll_context*) &this_monitor_context[1];  // set ctx->epoll
        ret = udrv_ll_dev_epoll_monitor_init(dev, config, this_monitor_context->epoll);  // [6] 4.4 set epoll ctx
    }
    if(ret){
        dev_error(dev, "init monitor context failed, error: %d\n", ret);
        udrv_mem_free(this_monitor_context);
    }
    else{
        *monitor_context = this_monitor_context;                                         // set devdata->monitor_ctx val
    }
    return ret;
}
```

4.3 setup custom monitor context to dev->devdata->monitor_context.

```blurtext
/* udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
static int udrv_ll_dev_custom_monitor_init(struct device* dev, struct custom_context* custom){
    int i;
    for(i = 0; i < MAX_INTERRUPT_DEVICES; i++){                                          // init null fd for monitoring
        custom->events[i].fd = -1;
    }
    return 0;
}
```

4.4 setup default epoll monitor context to dev->devdata->monitor_context.

```blurtext
/* udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
static int udrv_ll_dev_epoll_monitor_init(struct device* dev,                            // irq dev: reg state publisher
                                          struct udrv_interrupt_monitor_config config,
                                          struct epoll_context* epoll){
    int ret;
    int i;
    for(i = 0; i < MAX_INTERRUPT_DEVICES; i++){                                          // init null fd for monitoring
        epoll->events[i].fd = -1;
    }
    ret = udrv_epoll_init(&epoll->context, MAX_INTERRUPT_DEVICES);                                     // epoll_init
    if(ret){
        dev_error(dev, "init epoll context faied, error: %d\n", ret);
        return ret;
    }
    epoll->thread = udrv_plt_thread_create(config.thread_priority, epoll_monitor_thread_body, epoll);  // 4.3.1 set thd
    if(!epoll->thread){
        dev_error(dev, "create thread failed\n");
        udrv_epoll_deinit(&epoll->context);
        return ret;
    }
    return 0;
}
```

```blurtext
/* 4.3.1 udrv/platform/plt_adapter.c */
struct udrv_thread* udrv_plt_thread_create(int priority, void* (*start_routine)(void* args), void* args){
    if(user_interfaces_ptr && user_interfaces_ptr->udrv_plt_thread_create_fn){
        return user_interfaces_ptr->udrv_plt_thread_create_fn(priority, start_routine, args);  // 4.3.2
    }
    return default_interfaces_ptr->udrv_plt_thread_create_fn(priority, start_routine, args);   // 4.3.2
}

/* udrv/platform/plt_adapter.c: setup user/default interfaces fn */
static struct plt_interfaces default_plt_interfaces = {
    ...
    .udrv_plt_thread_create_fn = udrv_plt_default_thread_create,
    .udrv_plt_fd_event_add_fn = udrv_plt_default_fd_event_add,
    ...
};
static struct plt_interfaces* default_interfaces_ptr = &default_plt_interfaces;
static struct plt_interfaces* user_interfaces_ptr = NULL;

/* 4.3.2 udrv/platform/plt_adapter.c */
static struct udrv_thread* udrv_plt_default_thread_create(int priority, void* (*start_routine)(void* args), void* args){
    int ret;
    struct udrv_thread* thread;                                                 // [1] init udrv thread obj
    if(!start_routine){
        return NULL;
    }
    thread = udrv_mem_zalloc(sizeof(*thread));
    if(!thread){
        return NULL;
    }
    thread->user_start_routine = start_routine;                                 // [2] set thread job func
    thread->user_args = args;                                                   // [3] set thread job func args
    ret = pthread_create(&thread->thread, NULL, thread_body, thread);           // [4] posix pthread create
    if(ret){
        udrv_mem_free(thread);
        return NULL;
    }
    return thread;                                                              // return udrv thread ptr
}
```

<p style="margin-bottom: 20px;"></p>

5 interrupt controller driver supported operation registration.

```blurtext
static struct drv_operation interrupt_controller_opts = {
    .write = intrc_write,                                                       // write/mod the irq monitor config
    .read =  intrc_read,                                                        // read the irq monitor config
};
```

5.1 controller driver supported operations: read (get) & write (set) controller dev irq monitor cfg.

```blurtext
/* udrv/ll_interfaces/udrv_ll_driver_arguments.h: irq controller dev operation args format def */
struct interrupt_controller_args {
    int      is_register;                                                       // add/del monitor for irq state
    int      fd;                                                                // fd to be monitored
    int      operation;                                                         // op: read or write
    void (*interrupt_cb)(void* data);                                           // cb func when irq event happer
    void*    data;                                                              // cb func args
};
```

```blurtext
/* udrv/drivers/driver_interrupt_controller.c */
static int intrc_read(struct device* dev, void* args, void* data, int size){    // read size of data from dev cfg args
    struct interrupt_controller_devdata* devdata;
    struct interrupt_controller_args* intr_ctrl_args = (struct interrupt_controller_args*) args;
    if(!dev || !args || !data || size < sizeof(int)){
        pr_error("invalid arguments, func: %s, line: %d, dev: %p, args: %p, data: %p, size: %d\n",
                 __func__, __LINE__, dev, args, data, size);
        return -EINVAL;
    }
    devdata = udrv_dev_get_devdata(dev);                                        // get devdata of the irqdev to read
    if(!devdata){
        dev_error(dev, "not found the device data\n");
        return -EINVAL;
    }
    if(intr_ctrl_args->operation == INTR_CTRL_OPT_RD_NR_INTERRUPTS_MONITOR_STATE){  // ?
        *(int*)data = devdata->controller_monitor_type;                         // set & ret monitor type to child dev
        return 0;
    }
    dev_error(dev, "invalid read operation: %d\n", intr_ctrl_args->operation);
    return -EINVAL;
}
```

```blurtext
/* udrv/drivers/driver_interrupt_controller.c */
static int intrc_write(struct device* dev, void* args, void* data, int size){   // write/set dev monitor cfg by data
    struct interrupt_controller_devdata* devdata;
    struct interrupt_controller_args* intrcargs = (struct interrupt_controller_args*)args;
    int ret;
    if(!dev || !args){                                                          // if no irqdev or no arg assigned
        pr_error("invalid arguments, func: %s, line: %d, dev: %p, args %p\n",
                 __func__, __LINE__, dev, args);
        return -EINVAL;
    }
    if(intrcargs->fd < 0 || !intrcargs->interrupt_cb){                          // if no valid reg fd or no valid cb
        pr_error("invalid arguments, func: %s, line: %d, fd: %d, interrupt_cb %p\n",
                 __func__, __LINE__, intrcargs->fd, intrcargs->interrupt_cb);
        return -EINVAL;
    }
    devdata = udrv_dev_get_devdata(dev);                                        // get devdata of the irqdev to write
    if(!devdata){
        dev_error(dev, "not found the device data\n");
        return -EINVAL;
    }
    if(intrcargs->is_register){                                                 // register cb for reg fd of child dev
        ret = intrc_register_interrupt_event(dev, intrcargs);                   // 5.2
    }
    else{                                                                       // unregister cb for reg fd of child dev
        ret = intrc_unregister_interrupt_event(dev, intrcargs);                 // 5.3
    }
    return ret;
}
```

<p style="margin-bottom: 20px;"></p>

5.2 irq event register & unregister implementations.

```blurtext
/* udrv/drivers/driver_interrupt_controller.c */
static int intrc_register_interrupt_event(struct device* dev, struct interrupt_controller_args* args){
    int ret;
    struct interrupt_controller_devdata* devdata;                                       // get irqdev data
    devdata = udrv_dev_get_devdata(dev);
    ret = udrv_ll_dev_interrupt_monitor_add_event(devdata->monitor_context,             // 5.2.1
                                                  args->fd,                             // fd to monitor on
                                                  args->interrupt_cb,                   // 
                                                  args->data);                          // cb args
    if(ret){
        dev_error(dev, "add fd event failed, fd: %d, error: %d\n", args->fd, ret);
    }
    return ret;
}

/* udrv/drivers/driver_interrupt_controller.c */
static int intrc_unregister_interrupt_event(struct device* dev, struct interrupt_controller_args* args){
    int ret;
    struct interrupt_controller_devdata* devdata;
    devdata = udrv_dev_get_devdata(dev);
    ret = udrv_ll_dev_interrupt_monitor_del_event(devdata->monitor_context, args->fd);  // 5.2.2
    if(ret){
        dev_error(dev, "del fd event failed, fd: %d, error: %d\n", args->fd, ret);
    }
    return ret;
}
```

```blurtext
/* 5.2.1 udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
int udrv_ll_dev_interrupt_monitor_add_event(struct udrv_interrupt_monitor_context* monitor_context,
                                            int fd,
                                            void (*event_cb)(void* data),
                                            void* data){
    if(!monitor_context || fd < 0 || !event_cb){
        pr_error("invalid arguments, func: %s, line: %d, monitor_context %p, fd: %d, event_cb: %p\n",
                  __func__, __LINE__, monitor_context, fd, event_cb);
        return -EINVAL;
    }
    if(monitor_context->is_custom_monitor_method){
        return udrv_ll_dev_custom_add_event(monitor_context->dev,                       // 5.2.1.1
                                            monitor_context->custom,
                                            fd,
                                            event_cb,
                                            data);
    }
    else{                                                                               // 5.2.1.2
        return udrv_ll_dev_epoll_add_event(monitor_context->dev,                        // irq dev to monitor
                                           monitor_context->epoll,                      // epoll ctx
                                           fd,
                                           event_cb,
                                           data);
    }
}

/* 5.2.2 udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
int udrv_ll_dev_interrupt_monitor_del_event(struct udrv_interrupt_monitor_context* monitor_context, int fd){
    if(!monitor_context || fd < 0){
        pr_error("invalid arguments, func: %s, line: %d, monitor_context %p, fd: %d\n",
                  __func__, __LINE__, monitor_context, fd);
        return -EINVAL;
    }
    if(monitor_context->is_custom_monitor_method){
        return udrv_ll_dev_custom_del_event(monitor_context->dev, monitor_context->custom, fd);  // 5.2.2.1
    }
    else{
        return udrv_ll_dev_epoll_del_event(monitor_context->dev, monitor_context->epoll, fd);    // 5.2.2.2
    }
}
```

```blurtext
/* 5.2.1.1 udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
static int udrv_ll_dev_custom_add_event(struct device* dev,
                                        struct custom_context* custom,
                                        int fd,
                                        void (*event_cb)(void* data),
                                        void* data){
    int i;
    int ret = -EINVAL;
    for(i = 0; i < MAX_INTERRUPT_DEVICES; i++){                               // for each dev in subscriber list
        if(custom->events[i].fd < 0){
            custom->events[i].fd = fd;                                        // setup the fd to monitor
            ret = udrv_plt_fd_event_add(fd, UDRV_FD_EVENT_RD, event_cb, data, &(custom->events[i].handle));
            if(ret){
                dev_error(dev, "fd: %d, add custom fd event failed, error: %d\n", fd, ret);
                custom->events[i].fd = -1;
            }
            else{
                dev_info(dev, "fd: %d, add custom fd event success\n", fd);
            }
            break;
        }
    }
    return ret;
}

/* udrv/platform/plt_adapter.c: frontend platform api to add custom event */
int udrv_plt_fd_event_add(int fd,                                             // fd to monitor
                          int mode,                                           // trigger condition: r/w available
                          void (*event_cb)(void* data),                       // event trigger cb
                          void* data,                                         // cb args
                          void** handle){                                     // event handler ?
    if(user_interfaces_ptr && user_interfaces_ptr->udrv_plt_fd_event_add_fn){
        return user_interfaces_ptr->udrv_plt_fd_event_add_fn(fd, mode, event_cb, data, handle);
    }
    return default_interfaces_ptr->udrv_plt_fd_event_add_fn(fd, mode, event_cb, data, handle);
}

/* udrv/platform/plt_adapter.c: backend platform api to add custom event */
int udrv_plt_default_fd_event_add(int fd,
                                  int mode,
                                  void (*event_cb)(void* data),
                                  void* data,
                                  void** handle){
    printf("WARNING: function '%s' is not implemented\n", __func__);          // no implemented yet
    return -ENOSYS;
}

/* 5.2.1.2 udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
static int udrv_ll_dev_epoll_add_event(struct device* dev,                    // irq dev info
                                       struct epoll_context* epoll,           // epoll ctx
                                       int fd,                                // reg fd to monitor
                                       void (*event_cb)(void* data),          // user defined triggerred ev cb
                                       void* data){                           // cb arga
    int i;
    int ret = -EINVAL;;
    for(i = 0; i < MAX_INTERRUPT_DEVICES; i++){                               // for each dev in subscriber list
        if(epoll->events[i].fd < 0){                                          // setup epoll ctx event property
            epoll->events[i].fd = fd;                                         // set the reg fd to listen
            epoll->events[i].events = EPOLLIN | EPOLLET;                      // data ready to read, use edge trigger
            epoll->events[i].event_cb = udrv_ll_dev_epoll_event_cb;           // set udrv ll dev ev default cb?
            epoll->events[i].data[0] = (void*)event_cb;                       // set user defined cb
            epoll->events[i].data[1] = data;                                  // set user defined cb args
            ret = udrv_epoll_add_event(&epoll->context, &epoll->events[i]);   // -> add epoll event from udrv epoll ctx
            if(ret){
                dev_error(dev, "fd: %d, add epoll event failed, error: %d\n", fd, ret);
                epoll->events[i].fd = -1;
            }
            else{
                dev_info(dev, "fd: %d, add epoll event success\n", fd);
            }
            break;
        }
    }
    return ret;
}

/* -> udrv/utils/epoll.c */
int udrv_epoll_add_event(struct udrv_epoll_context* context, struct udrv_epoll_event* event){
    int ret;
    struct epoll_event ev;                                                    // init epoll_event to add
    if(!context || !event){
        pr_error("invalid arguments, context: %p, event: %p\n", context, event);
        return -EINVAL;
    }
    ev.events = event->events;                                                // set epoll flag: EPOLLIN | EPOLLET
    ev.data.ptr = event;                                                      // set epoll user data: ptr to udrv event
    ret = epoll_ctl(context->epoll_fd, EPOLL_CTL_ADD, event->fd, &ev);        // add event->fd & event to epoll obj
    if(ret){
        ret = -errno;
        pr_error("add epoll event failed, fd: %d, events: %d, error: %d\n", event->fd, event->events, ret);
    }
    return ret;
}
```

```blurtext
/* 5.2.2.2 udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
static int udrv_ll_dev_custom_del_event(struct device* dev,                  // irq dev: reg state publihsher
                                        struct custom_context* custom,       // custom ctx instance to del event from
                                        int fd){                             // del event that listening to fd
    int i;
    int ret = 0;
    for(i = 0; i < MAX_INTERRUPT_DEVICES; i++){
        if(fd > 0 && (custom->events[i].fd == fd)){
            ret = udrv_plt_fd_event_del(custom->events[i].handle);           // ->
            if(ret){
                dev_error(dev, "fd: %d, del custom fd event failed, error: %d\n", fd, ret);
            }
            else{
                dev_info(dev, "fd: %d, del custom fd event success\n", fd);
            }
            custom->events[i].fd = -1;
            break;
        }
    }
    return ret;
}

/* -> udrv/platform/plt_adapter.c */
int udrv_plt_fd_event_del(void* handle){
    if(user_interfaces_ptr && user_interfaces_ptr->udrv_plt_fd_event_del_fn){
        return user_interfaces_ptr->udrv_plt_fd_event_del_fn(handle);
    }
    return default_interfaces_ptr->udrv_plt_fd_event_del_fn(handle);
}

/* 5.2.2.2 udrv/ll_services/udrv_ll_interrupt_monitor_service.c */
static int udrv_ll_dev_epoll_del_event(struct device* dev,                    // irq dev: reg state publisher
                                       struct epoll_context* epoll,           // epoll instance to del event from
                                       int fd){                               // del event that listening to fd
    int i;
    int ret = 0;
    for(i = 0; i < MAX_INTERRUPT_DEVICES; i++){                               // for each subscriber dev of cur ev
        if(fd > 0 && (epoll->events[i].fd == fd)){                            // find the event related to given fd
            ret = udrv_epoll_del_event(&epoll->context, &epoll->events[i]);   // ->
            if(ret){
                dev_error(dev, "fd: %d, del epoll event failed, error: %d\n", fd, ret);
            }
            else{
                dev_info(dev, "fd: %d, del epoll event success\n", fd);
            }
            epoll->events[i].fd = -1;                                         // cleanup epoll event listening fd
            break;
        }
    }
    return ret;
}

/* -> udrv/utils/epoll.c */
int udrv_epoll_del_event(struct udrv_epoll_context* context, struct udrv_epoll_event* event){
    int ret;
    if(!context || !event){
        pr_error("invalid arguments, context: %p, event: %p\n", context, event);
        return -EINVAL;
    }
    ret = epoll_ctl(context->epoll_fd, EPOLL_CTL_DEL, event->fd, NULL);
    if(ret){
        ret = -errno;
        pr_error("del epoll event failed, fd: %d, events: %d, error: %d\n", event->fd, event->events, ret);
    }
    return ret;
}
```

<hr>

### # udrv.json example for interrupt controller device
the following shows a sample usage of udrv interrupt controller driver config, in which a device driver
config instance is included.

```blurtext
{
    "configs" : {                                                             // basic udrv dlib program config
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
        "compatible": "udrv interrupt controller",                            // level I irq controller driver

        "intr_ntio_presence": {                                               // ntio presence dev ?
            "compatible": "udrv uio interrupt device",                        // level II device driver
            "uioname": "uio_pipe_ntio_presence",                              // ?
            "auto-enable": false,                                             // ?
            "regs": {                                                         // reg config
                "storm": { "reg": "intr_storm.storm", "privatedata": "0x1" }  // if intr_storm.storm == 0x1
            }                                                                 // then ntio is present ?
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
```
