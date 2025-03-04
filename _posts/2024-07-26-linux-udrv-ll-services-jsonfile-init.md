---
layout: post
title: "udrv: udrv_ll_init -> return jsonfile path by jsonfile name (linux, udrv)"
author: "melon"
date: 1111-07-26 21:44
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
    ret = udrv_get_jsonfiles_dir(jsonfile);                           // *
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

2 udrv_get_jsonfile_dir: get jsonfile path by json file name.

```blurtext
/* udrv/ll_services/udrv_ll_basic_service.c */
static int udrv_get_jsonfiles_dir(char* jsonfile){
    int ret;
    char* json_real_pathname;
    ret = udrv_path_realpath(jsonfile, &json_real_pathname);              // udrv/utils/utils.h
    if(ret){
        pr_error("failed to get json file realpath, error: %d\n", ret);
        return ret;
    }
    ret = udrv_path_dirname(json_real_pathname, &json_dir);               // udrv/utils/utils.h
    if(ret){
        pr_error("failed to get json file dir, error: %d\n", ret);
        return ret;
    }
    pr_debug("json dir: %s\n", json_dir);
    return 0;
}
```
