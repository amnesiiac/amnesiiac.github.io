---
layout: post
title: "udrv: udrv_ll_init -> parse json config (linux, udrv)"
author: "melon"
date: 1111-08-01 21:47
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
    ret = udrv_ll_api_control_init();
    if(ret){
        pr_error("api control init failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_ll_parse_configs_in_json();                               // *
    if(ret){
        pr_error("parse configs in json failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_ll_parse_arguments(argc, argv);                           // *
    if(ret){
        pr_error("parse arguments failed, error: %d\n", ret);
        return ret;
    }
    udrv_ll_apply_print_configs();                                       // *
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

```blurtext
static int udrv_ll_parse_configs_in_json(){
    int ret;
    const char* value;
    if(!udrv_devnode_has_configs_node()){
        return 0;
    }
    ret = udrv_devnode_get_config_string("print_level", &value);
    if(ret == 0){
        print_level = udrv_ll_map_print_level((char*) value);
    }
    ret = udrv_devnode_get_config_string("print_verbose", &value);
    if(ret == 0){
        print_verbose = udrv_ll_map_print_verbose((char*) value);
    }
    ret = udrv_devnode_get_config_string("logfile", &value);
    if(ret == 0 && value){
        udrv_print_log_enable((char*) value);
    }
    ret = udrv_ll_parse_enable_api_config_in_json();
    if(ret < 0){
        return ret;
    }
    ret = udrv_ll_parse_device_filter_config_in_json();
    if(ret < 0){
        return ret;
    }
    return 0;
}
```
