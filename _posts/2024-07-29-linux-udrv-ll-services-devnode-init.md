---
layout: post
title: "udrv: udrv_ll_init -> devnode init (linux, udrv)"
author: "melon"
date: 1111-07-29 10:29
categories: "2024"
tags:
  - linux
  - driver 
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
    ret = udrv_devnode_init(jsonfile);                               // *
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

devnode recursively init, support multiple nested json files definition.

```blurtext
/* udrv/base/devnode.c */
int udrv_devnode_init(char* json_pathname){
    int ret;
    char* json_real_pathname;
    if(!json_pathname){
        pr_error("invalid argument, json_pathname: %p\n", json_pathname);
        return -EINVAL;
    }
    ret = udrv_path_realpath(json_pathname, &json_real_pathname);      // get json realpath (utils.c)
    if(ret){
        pr_error("can not get json file realpath, error: %d\n", ret);
        return ret;
    }
    ret = udrv_devnode_load_all_jsons(json_real_pathname);             // *
    udrv_str_free(json_real_pathname);                                 // free after usage (utils.c)
    if(ret){
        pr_error("load json failed, error: %d\n", ret);
        return ret;
    }
    ret = udrv_devnode_check();                                        // check name duplication between l1 & l2 devices
    if(ret){                                                           // controller dev & dev cannot has same devname
        pr_error("precheck json config failed, error: %d\n", ret);
        return ret;
    }
    return 0;
}
```

```blurtext
/* udrv/base/devnode.c */
static int udrv_devnode_load_all_jsons(char* json_pathname){
    int ret;
    char* json_dir;
    devnode_t* node = udrv_devnode_load_json(json_pathname);           // create devnode by json loaded
    if(!node){
        pr_error("load mail json failed\n");
        return -EINVAL;
    }
    ret = udrv_path_dirname(json_pathname, &json_dir);                 // get jsonfile dir to load subjson files (utils)
    if(ret){
        pr_error("get json file dir failed, error: %d\n", ret);
        return ret;
    }
    udrv_devnode_load_subjsons(json_dir, node);                        // load sub jsons
    json_root = node;                                                  // save json node entry
    return 0;
}
```

load json file by given json file path:

```blurtext
devnode_t* udrv_devnode_load_json(char* json_pathname){
    json_error_t error;
    devnode_t* node;
    if(!json_pathname){
        pr_error("invalid argument, json_pathname: %p\n", json_pathname);
        return NULL;
    }
    node = json_load_file(json_pathname, 0, &error);                   // see <jansson.h>
    if(!node){
        pr_error("Failed loading json file %s, source: %s, line: %d, column: %d\n",
                 json_pathname, error.text, error.source, error.line, error.column);
    }
    return node;
}
```

load all sub json files of given json file directory and devnode.

```blurtext
/* udrv/base/devnode.c */
static void udrv_devnode_load_subjsons(char* json_dir, devnode_t* node){
    const char* key1;
    const char* key2;
    devnode_t*  value1;
    devnode_t*  value2;
    devnode_t*  value3;
    char*       json_filename;
    char*       json_pathname;
    void*       tmp1;
    void*       tmp2;
    devnode_object_foreach_safe(node, tmp1, key1, value1){          // for each k,v in json node
        if(!udrv_is_import_key(key1)){                              // not import key, recursively load the sub node (1)
            udrv_devnode_load_subjsons(json_dir, value1);
            continue;
        }
        json_filename = (char*) json_string_value(value1);          // import key, buildup the subjson import path (2)
        json_pathname = udrv_path_combine(json_dir, json_filename);
        if(!json_pathname){
            pr_error("generate sub json file pathname failed, json_dir: (%p) %s, json_filename: (%p) %s\n",
                     json_dir, json_dir, json_filename, json_filename);
            return;
        }
        value2 = udrv_devnode_load_json(json_pathname);             // load the imported json data to devnode (2.1)
        udrv_str_free(json_pathname);
        if(!value2){
            continue;
        }
        udrv_devnode_load_subjsons(json_dir, value2);               // recursively load the import node's sub node (2.2)
        devnode_object_foreach_safe(value2, tmp2, key2, value3){    // for each k,v in sub json node
            udrv_devnode_set_object(node, key2, value3);
        }
        udrv_devnode_decref(value2);
        udrv_devnode_del_object(node, key1);                        // the node->key1 is done
    }
}
```

devnode check: (only support 2 logic level devices according to design)

```blurtext
/* udrv/base/devnode.c: check if l1 & l2 devname has duplication */
static int udrv_devnode_check(){
    int ret;
    ret = udrv_devnode_check_duplicated_devname();                              // *
    return ret;
}

/* udrv/base/devnode.c */
static int udrv_devnode_check_duplicated_devname(){
    int i;
    devnode_t*  level_1_node;
    devnode_t*  level_2_node;
    const char* level_1_node_name;
    const char* level_2_node_name;
    char**      devnames;
    int         nr_devname = 0;
    int         nr_device = 0;
    devnode_object_foreach(json_root, level_1_node_name, level_1_node{          // ret number of l1 + l2 devices
        if(!udrv_devnode_has_compatible(level_1_node)){                         // devnode has no compatible field, skip
            continue;
        }
        nr_device++;                                                            // add on l1 devices
        devnode_object_foreach(level_1_node, level_2_node_name, level_2_node){
            if(!udrv_devnode_has_compatible(level_2_node)){
                continue;
            }
            nr_device++;                                                        // add on l2 devices
        }
    }
    devnames = udrv_mem_zalloc(nr_device * sizeof(char*));                      // arr of all devname
    if(!devnames){
        pr_error("out of memory, %s\n", __func__);
        return -ENOMEM;
    }
    devnode_object_foreach(json_root, level_1_node_name, level_1_node){         // set l1 controller dev name
        if(udrv_devnode_has_compatible(level_1_node)){
            devnames[nr_devname++] = (char*) level_1_node_name;                 // add the l1 dev name into devnames
        }
    }
    /* iterate and check all level-2 devices (l1 is predefined by udrv, not possible to dup) */
    devnode_object_foreach(json_root, level_1_node_name, level_1_node){         // for each l1 devnode name 
        if(!udrv_devnode_has_compatible(level_1_node)){
            continue;
        }
        devnode_object_foreach(level_1_node, level_2_node_name, level_2_node){  // for each l2 devnode name
            if(!udrv_devnode_has_compatible(level_2_node)){
                continue;
            }
            for(i = 0; i < nr_devname; i++){                                    // if l2 devnode already existed
                if(udrv_str_equals((char*) level_2_node_name, devnames[i])){
                    pr_error("ERROR: found a duplicated devname: %s/%s\n", level_1_node_name, level_2_node_name);
                    udrv_mem_free(devnames);
                    return -EINVAL;
                }
            }
            devnames[i] = (char*) level_2_node_name;                            // add the new l2 dev name to devnames
            nr_devname++;
        }
    }
    udrv_mem_free(devnames);
    return 0;
}
```

helper functions for json object and udrv devnode json manipulation.

```blurtext
/* udrv/base/devnode.c */
static int udrv_is_import_key(const char* key){                                 // import key: startwith 'import'
    return udrv_str_startswith((char*) key, "import");
}

/* udrv/base/devnode.c */
int udrv_devnode_set_object(devnode_t* base, const char* name, devnode_t* node){
    return json_object_set(base, name, node);
}

/* udrv/base/devnode.c */
void udrv_devnode_decref(devnode_t* node){
    json_decref(node);
}

/* udrv/base/devnode.c */
int udrv_devnode_del_object(devnode_t* base, const char* name){
    return json_object_del(base, name);
}

/* udrv/base/devnode.c */
int udrv_devnode_has_compatible(devnode_t* node){
    return json_object_get(node, "compatible") != NULL;
}
```

helper macros: wrapper of <jansson.h> for each key-value pair in a given json file.

```blurtext
/* udrv/base/devnode.h */
typedef json_t devnode_t;                                              // devnode type def (jansson.h)

/* udrv/base/devnode.h */
#define devnode_object_foreach(devnode, key, value)                    \
        json_object_foreach(devnode, key, value)

/* udrv/base/devnode.h */
#define devnode_object_foreach_safe(devnode, tmp, key, value)          \   /* void* tmp */
        json_object_foreach_safe(devnode, tmp, key, value)
```
