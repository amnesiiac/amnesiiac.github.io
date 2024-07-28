---
layout: post
title: "udrv: udrv_ll_init -> platform interfaces module init (linux, udrv)"
author: "melon"
date: 2024-07-27 22:28
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
    ret = udrv_plt_module_init(interfaces);                          // *
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
    ret = udrv_ll_api_control_init();
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

2 setup default platform interfaces & user determined interfaces.

```text
/* udrv/ll_services/udrv_basic_ll_services */
int udrv_plt_module_init(struct plt_interfaces* interface){
    return udrv_plt_set_plt_interfaces(interface);
}

/* udrv/platform/plt_adapter.c */
static struct plt_interfaces* default_interfaces_ptr = &default_plt_interfaces;  // init default plt itf
static struct plt_interfaces* user_interfaces_ptr = NULL;

/* udrv/platform/plt_adapter.c: init user interface */
int udrv_plt_set_plt_interfaces(struct plt_interfaces* interface){               // set user plt itf
    user_interfaces_ptr = interface;
    return 0;
}

/* udrv/platform/plt_adapter.c: default plt itf def  */
static struct plt_interfaces default_plt_interfaces = {
    .udrv_plt_lock_new_fn = udrv_plt_default_lock_new,
    .udrv_plt_lock_free_fn = udrv_plt_default_lock_free,
    .udrv_plt_lock_fn = udrv_plt_default_lock,
    .udrv_plt_unlock_fn = udrv_plt_default_unlock,

    .udrv_plt_thread_create_fn = udrv_plt_default_thread_create,
    .udrv_plt_thread_destory_fn = udrv_plt_default_thread_destory,
    .udrv_plt_thread_get_current_fn = udrv_plt_default_get_current_thread,

    .udrv_plt_uio_open_fn = udrv_plt_default_uio_open,
    .udrv_plt_uio_close_fn = udrv_plt_default_uio_close,
    .udrv_plt_uio_get_fd_fn = udrv_plt_default_uio_get_fd,
    .udrv_plt_uio_enable_irq_fn = udrv_plt_default_uio_enable_irq,
    .udrv_plt_uio_disable_irq_fn = udrv_plt_default_uio_disable_irq,
    .udrv_plt_uio_get_mem_map_fn = udrv_plt_default_uio_get_mem_map,

    .udrv_plt_uio_generic_read_reg_ex_fn = udrv_plt_default_uio_generic_read_reg,
    .udrv_plt_uio_generic_write_reg_ex_fn = udrv_plt_default_uio_generic_write_reg,

    .udrv_plt_application_abort_fn = udrv_plt_default_application_abort,

    .udrv_plt_vprintf_fn = udrv_plt_default_vprintf,
    .udrv_plt_debug_vprintf_fn = udrv_plt_default_debug_vprintf,

    .udrv_plt_paddr_mmap_fn = udrv_plt_default_paddr_mmap,
    .udrv_plt_paddr_munmap_fn = udrv_plt_default_paddr_munmap,

    .udrv_plt_fd_event_add_fn = udrv_plt_default_fd_event_add,
    .udrv_plt_fd_event_del_fn = udrv_plt_default_fd_event_del,

    .udrv_plt_start_timer_fn = udrv_plt_default_start_timer,
    .udrv_plt_stop_timer_fn = udrv_plt_default_stop_timer,
};
```
