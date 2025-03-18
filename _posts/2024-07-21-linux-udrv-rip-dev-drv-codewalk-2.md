---
layout: post
title: "udrv: rip device driver code walk II (linux, udrv)"
author: "melon"
date: 1111-07-21 19:35
categories: "2024"
tags:
  - linux
  - driver
---

rip code walk I illustrate the main trunk of udrv infra, this article will focus on rip device driver.

<hr>

### # code walk of driver of rip device (from down to top)
1 driver basic struct definition & driver section setup:

```blurtext
/* udrv/drivers/driver_rip.c */
static struct driver rip_driver = {
    .name = "udrv rip driver",                                // drv name
    .compatible = "udrv rip device",                          // for matching of drv <-> drv 
    .init = rip_init,                                         // cb func called right before rip drv registed in udrv
    .probe = rip_probe,                                       // cb func called when add new rip dev to rip drv's bus
    .opt = &rip_opts,                                         // supported operations
};

/* udrv/drivers/driver_rip.c */
udrv_device_driver(rip_driver);                               // set rip_driver into device_driver section
```

<p style="margin-bottom: 20px;"></p>

2 rip device backend data structure definitions:

```blurtext
/* udrv/drivers/driver_rip.c: device core data structure devdata */
struct rip_devdata {
    struct list                           filefields;         // list of file field config associated with rip
    int                                   cache_enabled;      // enable cache or not
    struct udrv_cache_group*              group;              // ptr to udrv_cache_group
    struct udrv_device_following_context* following_context;  // ptr to the ctx that rip is following
    struct list                           forced_values;      // ...
    char*                                 raw_pathname;       // path to raw data associated with rip
    struct list                           raw_configs;        // list of raw data config (define how to r/w each field)
};
```

```blurtext
/* udrv/drivers/driver_rip.c: rip device "forced" field value entry structure */
struct forced_value_entry {
    char*     fieldname;
    char*     value;
    struct    list link;
};
```

```blurtext
/* udrv/drivers/driver_rip.c: rip device "raw" field value entry structure */
struct raw_value_config {
    char*     fieldname;
    int       offset;
    int       length;
    struct    list link;
};
```

```blurtext
/* udrv/ll_services/udrv_ll_dev_following.h: udrv device following context data structure */
struct udrv_device_following_context {
    struct device* following_dev;                     // ptr to the dev following
    void* notification_handle;                        // ptr to the udrv_notification struct
};
```

<p style="margin-bottom: 20px;"></p>

3 rip driver supported operations:

```blurtext
/* udrv/drivers/driver_rip.c */
static struct drv_operation rip_opts = {
    .read = rip_read,                                                        // read
    .write = rip_write,                                                      // write
};

static int rip_read(struct device* dev, void* args, void* data, int size){
    return rip_read_write(dev, args, data, size, 0);                         // 3.1
}

static int rip_write(struct device* dev, void* args, void* data, int size){
    return rip_read_write(dev, args, data, size, 1);                         // 3.1
}
```

3.1 core function for rip device read & write operation.

```blurtext
static int rip_read_write(struct device* dev, void* args, void* data, int size, int is_write){
    struct rip_devdata*         devdata;                                     // rip devdata
    struct filefield*           field;                                       // file field
    struct raw_value_config*    raw_config;                                  // raw config
    struct rip_dev_args*        ripargs = (struct rip_dev_args*) args;       // dev args
    DECLARE_FILE_CONTROLLER_ARGS(fileargs);
    int                         rwsize;                                      // size to r/w
    int                         ret;                                         // return val
    if(!dev || !args || !data || size < 1){
        pr_error("invalid arguments, func: %s, line: %d, dev: %p, args: %p, data: %p, size: %d\n",
                 __func__, __LINE__, dev, args, data, size);
        return -EINVAL;
    }
    devdata = udrv_dev_get_devdata(dev);                                     // get devdata of the dev operate on
    if(!devdata){
        dev_error(dev, "not found the device data\n");
        return -EINVAL;
    }
    if(!ripargs->fieldname){
        dev_error(dev, "invalid arguments, did not provide the 'fieldname'\n");
        return -EINVAL;
    }
    rwsize = size;
    if(rip_has_forced_value(dev, devdata, ripargs->fieldname)){              // 3.2: get ret from forced val if exist
        return rip_read_forced_value(dev, devdata, ripargs->fieldname, data, rwsize);
    }
    raw_config = rip_get_raw_configs(dev, devdata, ripargs->fieldname);      // 3.4: get path & offset & rwsize by raw
    if(raw_config){
        fileargs.pathname = devdata->raw_pathname;
        fileargs.offset = raw_config->offset;
        rwsize = raw_config->length > size ? size : raw_config->length;
    }
    else{                                                                    // 3.5: get field info by filefields
        field = udrv_ll_dev_find_filefield(&(devdata->filefields), ripargs->fieldname);
        if(!field){
            dev_error(dev, "does not support the rip field: %s\n", ripargs->fieldname);
            return -ENODEV;
        }
        fileargs.pathname = field->pathname;
    }
    if(is_write){                                                            // write operation
        ret = udrv_ll_dev_write(dev->parent, &fileargs, data, rwsize);       // 3.6: invoke parent dev drv's write
        udrv_dev_io_usage_update(dev, 1, ret < 0);
        if(ret <= 0){
            dev_error(dev, "write file '%s' failed, error: %d\n", field->pathname, ret);
            return ret;
        }
    }
    else{                                                                    // read operation
        if(devdata->cache_enabled){                                          // 3.7: read from cache if cache enabled
            ret = rip_read_from_cache(devdata->group, field->pathname, data, rwsize);
            if(ret > 0){
                dev_debug(dev, "found data in cache, fieldname: %s\n", ripargs->fieldname);
                return ret;
            }
        }
        ret = udrv_ll_dev_read(dev->parent, &fileargs, data, rwsize);        // 3.8: invoke parent dev drv's read
        udrv_dev_io_usage_update(dev, 0, ret < 0);
        if(ret <= 0){
            dev_error(dev, "read file '%s' failed, error: %d\n", field->pathname, ret);
            return ret;
        }
    }
    if(devdata->cache_enabled){                                              // 3.9: update dev's cache after r/w
        if(rip_update_cache(devdata->group, field->pathname, data, ret) == 0){
            dev_debug(dev, "save fieldname: %s data to cache\n", ripargs->fieldname);
        }
    }
    return ret;
}
```

3.2 judge if rip device data has forced value named as fieldname:

```blurtext
/* udrv/drivers/driver_rip.c */
static int rip_has_forced_value(struct device* dev, struct rip_devdata* devdata, const char* fieldname){
    struct forced_value_entry* entry;
    list_for_each_entry(entry, &(devdata->forced_values), link){
        if(udrv_str_equals(entry->fieldname, fieldname)){
            return 1;
        }
    }
    return 0;
}
```

3.3 get forced value of fieldname by iteration over the dev->devdata:

```blurtext
/* udrv/drivers/driver_rip.c */
static int rip_read_forced_value(struct device* dev, struct rip_devdata* devdata,
                                 const char* fieldname, void* data, int size){
    struct forced_value_entry* entry;
    list_for_each_entry(entry, &(devdata->forced_values), link){
        if(udrv_str_equals(entry->fieldname, fieldname)){
            dev_info(dev, "found the forced value for fieldanme: %s, value: %s\n", fieldname, entry->value);
            return udrv_str_n_safe_copy(entry->value, udrv_str_length(entry->value), data, size);
        }
    }
    return 0;
}
```

3.4 return raw value config of dev by given fieldname:

```blurtext
/* udrv/drivers/driver_rip.c */
static struct raw_value_config* rip_get_raw_configs(struct device* dev,
                                                    struct rip_devdata* devdata,
                                                    const char* fieldname){
    struct raw_value_config* config;
    list_for_each_entry(config, &(devdata->raw_configs), link){
        if(udrv_str_equals(config->fieldname, fieldname)){
            return config;
        }
    }
    return NULL;
}
```

3.5 return filefield structure by fieldname from the given filefield list:

```blurtext
/* udrv/ll_services/udrv_ll_field_service.c */
struct filefield* udrv_ll_dev_find_filefield(struct list* filefields, const char* fieldname){
    struct filefield* field = NULL;
    if(!filefields || !fieldname){
        return NULL;
    }
    list_for_each_entry(field, filefields, link){
        if(udrv_str_equals(field->name, fieldname)){
            return field;
        }
    }
    return NULL;
}
```

3.6 invoke write operation of given dev->drv:

```blurtext
/* udrv/ll_services/udrv_ll_general_io_service.c */
int udrv_ll_dev_write(struct device* dev, void* args, void* data, int size){
    if(dev && dev->drv && dev->drv->opt && dev->drv->opt->write){
        return dev->drv->opt->write(dev, args, data, size);
    }
    pr_error("No implementation for the WRITE operation or no driver serves this device\n");
    return -EINVAL;
}
```

3.7 read data from cache group, return the data length, 0 means cache miss:

```blurtext
/* udrv/drivers/driver_rip.c */
static int rip_read_from_cache(struct udrv_cache_group* group, char* pathname, void* data, int size){
    struct udrv_cache* cache  = udrv_ll_cache_group_find_cache(group, pathname);
    if(cache){
        return udrv_ll_cache_data_dump(cache, data, size);
    }
    return 0;
}
```

3.8 invoke read operation of given dev->drv:

```blurtext
/* udrv/ll_services/udrv_ll_general_io_service.c */
int udrv_ll_dev_read(struct device* dev, void* args, void* data, int size){
    if(dev && dev->drv && dev->drv->opt && dev->drv->opt->read){
        return dev->drv->opt->read(dev, args, data, size);
    }
    pr_error("No implementation for the READ operation or no driver serves this device\n");
    return -EINVAL;
}
```

3.9 update cache group entry identified by pathname:

```blurtext
/* udrv/drivers/driver_rip.c */
static int rip_update_cache(struct udrv_cache_group* group, char* pathname, void* data, int size){
    struct udrv_cache* cache;
    cache = udrv_ll_cache_group_find_cache(group, pathname);         // set cache by pathname in cache group
    if(cache){
        return 0;
    }
    cache = udrv_ll_cache_alloc_with_data(pathname, data, size);     // set cache by data size
    if(!cache){
        return -ENOMEM;
    }
    return udrv_ll_cache_group_add_cache(group, cache);              // add cache to dev's cache group
}
```

3.10 fill info to io usage statistics:

```blurtext
int udrv_dev_io_usage_update(struct device* dev, int is_write, int is_failure){
    struct device_op_usage* usage;
    if(!dev){
        return -EINVAL;
    }
    usage = is_write ? &dev->usage[UDRV_USAGE_TYPE_IO_WRITE] : &dev->usage[UDRV_USAGE_TYPE_IO_READ];
    if(is_failure){
        usage->failure++;
    }
    else{
        usage->success++;
    }
    usage->is_updated = 1;
    return 0;
}
```

<p style="margin-bottom: 20px;"></p>

4 init function invoked when register rip driver into device driver section:

```blurtext
/* udrv/drivers/driver_rip.c */
static int rip_init(struct driver* drv){
    drv->bus = udrv_bus_find_bus(UDRV_FILE_BUS);              // set the rip driver's bus as FILE BUS
    return 0;
}
```

<p style="margin-bottom: 20px;"></p>

5 probe function invoked when add new rip dev to rip driver's bus:

```blurtext
/* udrv/drivers/driver_rip.c */
static int rip_probe(struct device* dev){
    int ret = 0;
    struct rip_devdata* devdata;                                                // devdata of newly-add dev
    devdata = udrv_mem_zalloc(sizeof(*devdata));                                // zalloc mem for dev->devdata
    if(!devdata){
        dev_error(dev, "can not get memory for device data\n");
        return -ENOMEM;
    }
    list_init(&(devdata->filefields));                                          // init devdata property list node
    list_init(&(devdata->forced_values));
    list_init(&(devdata->raw_configs));
    if(udrv_ll_dev_has_property(dev, "forced")){                                // if dev added has force field in json
        dev_debug(dev, "collect forced values\n");
        ret = rip_collect_forced_value(dev, devdata);                           // 6: update force_value lst by dev
    }
    else if(udrv_ll_dev_has_property(dev, "raw")){
        dev_debug(dev, "collect raw configs\n");
        ret = rip_collect_raw_configs(dev, devdata);                            // 7: update raw config lst by dev
    }
    else if(udrv_ll_dev_has_property(dev, "files")){
        dev_debug(dev, "collect files from json\n");
        ret = udrv_ll_dev_collect_filefield_from_json(dev, &(devdata->filefields));  // 8: update filefield lst by json
    }
    else{
        dev_debug(dev, "collect all files from given path\n");
        ret = udrv_ll_dev_collect_filefield_from_path(dev, &(devdata->filefields));  // 9: update filefield lst by path
    }
    if(ret < 0){
        dev_error(dev, "collect files failed, error: %d\n", ret);
        udrv_mem_free(devdata);
        return ret;
    }
    if(udrv_ll_dev_property_is_true(dev, "cache-enabled")){                          // if cache-enabled is set
        devdata->cache_enabled = 1;
        dev_info(dev, "data cache is enabled\n");
        devdata->group = udrv_ll_cache_group_init(rip_cache_tag_compare, NULL);      // 10: init devdata->cache group
        if(!devdata->group){
            dev_error(dev, "create cache group failed\n");
            udrv_mem_free(devdata);
            return -ENOMEM;
        }
    }
    udrv_dev_set_devdata(dev, devdata);                  // ref: base/device.h, set dev->devdata
    if(udrv_ll_dev_device_following_has_configs(dev)){   // ll_services/udrv_ll_device_following_service
        ret = udrv_ll_dev_device_following_init(         // 11: init ctx & register cb for subscribed event
            dev,
            &devdata->following_context,                 // pass addr to set dev->devdata following ctx
            rip_on_following_device_state_changed,       // 12: cb when the state of the dev followed changed
            dev
        );
        if(ret){
            dev_error(dev, "follow device failed, error: %d\n", ret);
            udrv_mem_free(devdata);
            return ret;
        }
    }
    return 0;
}
```

```blurtext
/* udrv/drivers/driver_rip.c: compare cache tag (a.k.a field pathname) */
static int rip_cache_tag_compare(void* pathname1, void* pathname2){
    if(pathname1 && pathname2){
        return pathname1 == pathname2;
    }
    return 0;
}
```

<p style="margin-bottom: 20px;"></p>

6: collect all force value from certain dev's devnode as drv's force_value list entry:

```blurtext
/* udrv/drivers/driver_rip.c */
static int rip_collect_forced_value(struct device* dev, struct rip_devdata* devdata){
    const char*                   fieldname;
    devnode_t*                    map_nodes;
    devnode_t*                    map_node;
    struct forced_value_entry*    entry;
    map_nodes = udrv_devnode_get_object(dev->devnode, "forced");        // get dev json node forced field
    devnode_object_foreach(map_nodes, fieldname, map_node){             // for each kv pair of devnode force field
        entry = udrv_mem_zalloc(sizeof(*entry));                        // zalloc for force data obj
        if(!entry){
            return -ENOMEM;
        }
        entry->fieldname = udrv_str_dup((char*) fieldname);             // setup force value obj
        entry->value = udrv_str_dup((char*) json_string_value(map_node));
        dev_debug(dev, "new forced value entry, fieldname: %s, value: %s\n",
                  entry->fieldname, entry->value);
        list_append(&(devdata->forced_values), &(entry->link));         // add dev's force value obj to drv's force list
    }
    return 0;
}
```

<p style="margin-bottom: 20px;"></p>

7 collect all raw config from certain dev's devnode as drv's raw config list entry:

```blurtext
/* udrv/drivers/driver_rip.c */
static int rip_collect_raw_configs(struct device* dev, struct rip_devdata* devdata){
    const char*                 fieldname;
    devnode_t*                  map_nodes;
    devnode_t*                  map_node;
    int                         ret;
    struct raw_value_config*    config;
    devdata->raw_pathname = udrv_ll_dev_property_path(dev, "path");
    if(!devdata->raw_pathname){
        return -EINVAL;
    }
    map_nodes = udrv_devnode_get_object(dev->devnode, "raw");
    devnode_object_foreach(map_nodes, fieldname, map_node){
        config = udrv_mem_zalloc(sizeof(*config));
        if(!config){
            return -ENOMEM;
        }
        config->fieldname = udrv_str_dup((char*) fieldname);
        ret = udrv_devnode_scan(map_node, "{s:i, s:i}", "offset", &config->offset, "length", &config->length);
        if(ret){
            pr_error("error on parse raw config, name: %s, error: %d\n", fieldname, ret);
            return ret;
        }
        dev_debug(dev, "new raw config, fieldname: %s, offset: %d, length: %d\n",
                  config->fieldname, config->offset, config->length);
        list_append(&(devdata->raw_configs), &(config->link));
    }
    return 0;
}
```

<p style="margin-bottom: 20px;"></p>

8: buildup filefield list by json.

```blurtext
/* udrv/ll_services/udrv_ll_field_services.c */
int udrv_ll_dev_collect_filefield_from_json(struct device* dev, struct list* filefields_out){
    struct creating_filefield_context context;
    if(!dev || !filefields_out){
        return -EINVAL;
    }
    context.fields = filefields_out;
    context.path = (char*) udrv_ll_dev_property_path(dev, "path");
    return udrv_devnode_foreach_filefield(dev->devnode, create_filefield_helper, &context);
}
```

<p style="margin-bottom: 20px;"></p>

9: buildup filefield list by "path" key in udrv.json.

```blurtext
/* udrv/ll_services/udrv_ll_field_services.c */
int udrv_ll_dev_collect_filefield_from_path(struct device* dev, struct list* filefields_out){
    char* path;
    int ret;
    struct creating_filefield_context context;                                // core data ctx init
    if(!dev || !filefields_out){
        return -EINVAL;
    }
    path = (char*) udrv_ll_dev_property_path(dev, "path");                    // 9.1 return the "path" value
    if(!path){
        return -EINVAL;
    }
    context.fields = filefields_out;                                          // set ctx property
    context.path = path;
    ret = udrv_path_iterate_file(path, create_filefield_helper, &context);    // 9.2 iterate & handle each f under path
    udrv_str_free(path);                                                      //     to set the rest ctx property
    return ret;
}
```

9.1 return the udrv.json key property "path" corresponding value in char* format.

```blurtext
/* udrv/ll_services/udrv_ll_basic_service.c */
char* udrv_ll_dev_property_path(struct device* dev, const char* path_property_name){
    const char* path;
    if(!dev || !path_property_name){
        pr_error("invalid arguments, func: %s, line: %d, dev: %p, path_property_name: %p\n",
                 __func__, __LINE__, dev, path_property_name);
        return NULL;
    }
    path = udrv_ll_dev_property_string(dev, path_property_name);
    if(!path){
        dev_error(dev, "not found the 'path' property\n");
        return NULL;
    }
    return udrv_ll_dev_candidate_path(dev, path);
}

/* udrv/ll_services/udrv_ll_basic_service.c */
const char* udrv_ll_dev_property_string(struct device* dev, const char* property_name){
    if(!dev || !property_name){
        return NULL;
    }
    if(!dev->devnode){
        return NULL;
    }
    return udrv_devnode_get_string(dev->devnode, property_name);
}
```
```blurtext
/* udrv/base/devnode.c */
const char* udrv_devnode_get_string(devnode_t* node, const char* name){
    devnode_t* value = json_object_get(node, name);
    return udrv_devnode_to_string(value);
}

/* udrv/base/devnode.c */
const char* udrv_devnode_to_string(devnode_t* node){
    return node ? json_string_value(node) : NULL;
}
```

9.2 iterate over all candidate file under "path", apply the handle_fn for each of them.

```blurtext
/* udrv/utils/utils.c */
int udrv_path_iterate_file(char* path, int (*handle_fn)(const char* filename, void* data), void* data){
    DIR* dir;
    struct dirent* dir_entry;
    char* pathname;
    int ret = 0;
    if(!path || !handle_fn){
        return -EINVAL;
    }
    dir = opendir(path);
    if(!dir){
        return -ENOENT;
    }
    for(;;){                                                     // iterate over the obj under the path
        dir_entry = readdir(dir);
        if(!dir_entry){                                          // ignore not dir_entry
            break;
        }
        if(udrv_str_equals(dir_entry->d_name, ".")){             // ignore .
            continue;
        }
        if(udrv_str_equals(dir_entry->d_name, "..")){            // ignore ..
            continue;
        }
        // we don't use 'dir_entry->d_type' as some fs does not support the 'd_type' very well
        pathname = udrv_path_combine(path, dir_entry->d_name);   // buildup reg file absolute path
        if(udrv_is_candidate_file(pathname)){                    // if pathname is a candidate
            ret = handle_fn(dir_entry->d_name, data);            // 9.3 update filepath to newly add rip drv ctx
        }
        udrv_str_free(pathname);
        if(ret){                                                 // if certain file cannot handled correctly
            break;                                               // break loop & ret
        }
    }
    closedir(dir);
    return ret;
}

/* udrv/utils/utils.c */
static int udrv_is_candidate_file(char* pathname){                // judge whether a file can be a candidate
    struct stat statbuf;
    if(stat(pathname, &statbuf)){                                 // if stat fail (ret 1), not a candidate file
        return 0;
    }
    return S_ISREG(statbuf.st_mode) || S_ISCHR(statbuf.st_mode);  // judge if a regular or char file
}
```

9.3 handler function for processing each file under udrv.json "path".

```blurtext
/* udrv/ll_services/udrv_ll_field_service.c */
static int create_filefield_helper(const char* name, void* data){
    struct creating_filefield_context* context = (struct creating_filefield_context*) data;
    struct filefield* field = udrv_field_create_filefield((char*) name);   // init filefield struct obj
    if(field){
        field->pathname = udrv_path_combine(context->path, field->name);   // set fielfield obj pathname
        list_append(context->fields, &(field->link));                      // update the filefield link to ctx field lst
    }
    return 0;
}

/* udrv/base/field.c */
struct filefield* udrv_field_create_filefield(char* name){                 // return filefield struct obj
    struct filefield* f = udrv_mem_zalloc(sizeof(*f));
    if(f){
        f->name = udrv_str_dup(name);                                      // set fielfield obj name
        list_init(&(f->link));
    }
    return f;
}
```

<p style="margin-bottom: 20px;"></p>

10: init cache group obj, and return the pointer to the group instance to the caller.

```blurtext
/* udrv/ll_services/udrv_ll_cache_services.c */
struct udrv_cache_group* udrv_ll_cache_group_init(int (*tag_compare_fn)(void* tag1, void* tag2),
                                                  void (*cache_free_fn)(struct udrv_cache* cache)){
    struct udrv_cache_group* group;                                     // init cache group
    if(!tag_compare_fn){
        pr_error("invalid arguments, func: %s, line: %d, tag_compare_fn: %p\n",
                 __func__, __LINE__, tag_compare_fn);
        return NULL;
    }
    group = udrv_mem_zalloc(sizeof(*group));                            // alloc mem for cache group
    if(!group){
        pr_error("invalid arguments, func: %s, line: %d, can not alloc memory for udrv cache group\n",
                 __func__, __LINE__);
        return NULL;
    }
    group->tag_compare_fn = tag_compare_fn;                             // set cache tag cmp to group obj
    group->cache_free_fn = cache_free_fn;                               // set cache free to group obj
    list_init(&(group->caches));                                        // init group cache list
    return group;
}

/* udrv/ll_services/udrv_ll_cache_services.c */
struct udrv_cache_group {
    int (*tag_compare_fn)(void *tag1, void *tag2);
    void (*cache_free_fn)(struct udrv_cache *cache);
    struct list caches;                                                 // d-linked list data cache
};
```

<p style="margin-bottom: 20px;"></p>

11: init dev to follow-up, init dev-follow ctx,

```blurtext
/* udrv/ll_services/udrv_ll_device_following_service.c */
int udrv_ll_dev_device_following_init(struct device* dev,
                                      struct udrv_device_following_context** following_context,
                                      void (*on_following_device_state_changed)(void* notify_data, void* user_data),
                                      void* user_data){
    int ret;
    struct device* following_dev;
    char* following_devname;
    struct udrv_device_following_context* this_following_context;                    // following dev ctx of cur dev
    void (*fn)(void*, void*) = on_following_device_state_changed;                    // set onchange callback as fn
    if(!dev || !following_context){
        pr_error("invalid arguments, func: %s, line: %d, dev: %p, following_context: %p\n",
                 __func__, __LINE__, dev, following_context);
        return -EINVAL;
    }
    following_devname = (char*) udrv_ll_dev_property_string(dev, "following");       // set following dev name
    if(!following_devname){
        dev_error(dev, "get the following device name failed\n");
        return -EINVAL;
    }
    ret = udrv_ll_dev_get_device_instance(following_devname, &following_dev);        // ret obj of the dev following
    if(ret){
        dev_error(dev, "can not found the following device: %s instance\n", following_devname);
        return ret;
    }
    this_following_context = udrv_mem_zalloc(sizeof(*this_following_context));       // alloc mem for ctx
    if(!this_following_context){
        dev_error(dev, "can not alloc memory for creating 'this_following_context'\n");
        return -ENOMEM;
    }
    this_following_context->following_dev = following_dev;                           // set ctx->dev = following dev
    if(!fn){                                                                         // set default cb if fn not set
        fn = udrv_ll_dev_device_following_default_following_device_state_changed_cb;
    }
    ret = udrv_ll_dev_subscribe_event_notification(                                  // 12: subscribe event
        following_dev, fn, user_data, &(this_following_context->notification_handle)
    );
    if(ret){
        dev_error(dev, "failed to subscribe event notification to the following device: %s, error: %d\n",
                  following_devname, ret);
        udrv_mem_free(this_following_context);
    }
    *following_context = this_following_context;
    return ret;
}
```

<p style="margin-bottom: 20px;"></p>

12: subscribe event notification of the dev followed, drv->probe only activate data placeholders, callback
registrations, rather than actually triggering the notification operation:
(udrv_dev_invoke_notification ─> udrv_invoke_notification ─> notification_fn).

```blurtext
/* udrv/ll_services/udrv_ll_interrupt_service.c */
int udrv_ll_dev_subscribe_event_notification(
    struct device* dev,                                              // dev to follow up
    void (*notification_fn)(void* notify_data, void* user_data),     // cb to notify cur dev when event occur
    void* user_data,                                                 // cur dev
    void** handle){                                                  // ptr to notification
    return udrv_dev_subscribe_event_notification(dev, notification_fn, user_data, handle);
}

/* udrv/base/device.c */
int udrv_dev_subscribe_event_notification(
    struct device* dev, void (*notification_fn)(void* notify_data, void* user_data), void* user_data, void** handle){
    return udrv_dev_subscribe_event_notification_on_thread(dev, notification_fn, user_data, handle, 0);
}

/* udrv/base/device.c */
static int udrv_dev_subscribe_event_notification_on_thread(          // protect needed when probe executed concurrently
    struct device* dev,                                              // a) kernel concurrent device init
    void (*notification_fn)(void* notify_data, void* user_data),     // b) concurrent hot-plug dev
    void* user_data,                                                 // c) dynamically added dev in virtual env
    void** handle,
    int thread_affinity){
    int ret;
    if(!dev || !notification_fn || !handle){
        return -EINVAL;
    }
    udrv_dev_lock(dev);                                                      // 12.1: lock dev: ensure state consist
    if(thread_affinity){                                                     // enable thread affinity
        ret = udrv_subscribe_thread_affinity_notification(
            &(dev->event_notifications), notification_fn, user_data, handle
        );
    }
    else{                                                                    // disable thread affinity
        ret = udrv_subscribe_notification(
            &(dev->event_notifications), notification_fn, user_data, handle
        );
    }
    if(ret == 0){
        dev->nr_event_subscriber++;
    }
    udrv_dev_unlock(dev);                                                    // 12.1: unlock dev obj
    return ret;
}
```

```blurtext
/* udrv/utils/notification.c */
int udrv_subscribe_thread_affinity_notification(struct udrv_notification_chain* chain,
                                                void (*notification_fn)(void* notify_data, void* user_data),
                                                void* user_data,
                                                void** handle){
    unsigned long thread_id;                                                 // get current thread id
    udrv_plt_thread_get_current(&thread_id);
    return udrv_subscribe_notification_on_thread(chain, thread_id, notification_fn, user_data, handle);
}
```

```blurtext
/* udrv/utils/notification.c */
int udrv_subscribe_notification(struct udrv_notification_chain* chain,
                                void (*notification_fn)(void* notify_data, void* user_data),
                                void* user_data,
                                void** handle){
    return udrv_subscribe_notification_on_thread(chain, NON_AFFINITY_THREAD_ID, notification_fn, user_data, handle);
}
```

```blurtext
/* udrv/utils/notification.c */
static int udrv_subscribe_notification_on_thread(struct udrv_notification_chain* chain,
                                                 unsigned long thread_id,
                                                 void (*notification_fn)(void* notify_data, void* user_data),
                                                 void* user_data,
                                                 void** handle){
    struct udrv_notification* notification;                                  // 12.2: init notification
    if(!chain || !notification_fn || !handle){
        return -EINVAL;
    }
    notification = udrv_mem_zalloc(sizeof(*notification));                   // alloc notification mem
    if(!notification){
        return -ENOMEM;
    }
    notification->notification_fn = notification_fn;                         // set notification
    notification->user_data = user_data;
    notification->notify_state = UDRV_NOTIFY_STATE_INIT;
    notification->thread_id = thread_id;                                     // set notify thread id
    list_append(&(chain->notifications), &(notification->link));             // add to cur drv's notification chain
    *handle = notification;                                                  // set dev following ctx->handle
    return 0;
}
```

12.1: lock & unlock utiliy used in udrv_dev_subscribe_event_notification_on_thread function.

```blurtext
/* udrv/base/device.c */
int udrv_dev_lock(struct device* dev){
    int ret;
    if(!dev){
        return -EINVAL;
    }
    ret = udrv_plt_lock(dev->lock);        // use platform user interface lock_fn to lock the dev
    if(ret == 0){
        dev->locked = 1;                   // set dev->locked as 1, avoid self re-acquire the lock
    }
    return ret;
}

/* udrv/platform/plt_adapter.c */
int udrv_plt_lock(struct udrv_lock* lock){
    if(user_interfaces_ptr && user_interfaces_ptr->udrv_plt_lock_fn){        // if customized itf lock_fn available
        return user_interfaces_ptr->udrv_plt_lock_fn(lock);
    }
    return default_interfaces_ptr->udrv_plt_lock_fn(lock);                   // use default lock_fn
}

/* udrv/platform/plt_adapter.c */
static int udrv_plt_default_lock(struct udrv_lock* lock){
    return lock ? pthread_mutex_lock(&lock->lock) : -EINVAL;
}
```

12.2: udrv notification payload data structure definition.

```blurtext
/* udrv/utils/notification.h */
struct udrv_notification {
    struct list link;
    void (*notification_fn)(void* notify_data, void* user_data);             // cb
    void* user_data;                                                         // dev who recv the notification
    int notify_state;
    unsigned long thread_id;                                                 // thread id
};
```

<p style="margin-bottom: 20px;"></p>

13: callback function definition: invoked when the state of the dev followed changed.

```blurtext
/* udrv/drivers/driver_rip.c */
void rip_on_following_device_state_changed(void* notify_data, void* user_data){
    struct device* dev = (struct device*) user_data;
    struct device_notification_data* notification_data = (struct device_notification_data*)notify_data;  // 13.1
    udrv_ll_dev_device_following_default_following_device_state_changed_cb(notify_data, user_data);      // 13.2
    if(!udrv_dev_is_ready(notification_data->dev)){
        rip_drop_cache_if_need(dev);                                                                     // 13.3
    }
}
```

13.1: notification payload structure definition.

```blurtext
/* udrv/base/device.h */
struct device_notification_data {
    struct device*    dev;                                                   // dev who send the notification
    int               state;                                                 // dev state
    unsigned int      events;                                                // event id
    void*             data;
};
```

13.2: udrv default dev following state change callback definition.

```blurtext
/* udrv/ll_services/udrv_ll_device_following_service.c */
void udrv_ll_dev_device_following_default_following_device_state_changed_cb(void* notify_data, void* user_data){
    struct device* dev = (struct device*) user_data;                                     // the subscriber dev
    struct device_notification_data* notification_data = (struct device_notification_data*)notify_data;  // payload 
    if(!notify_data || !user_data){
        pr_error("invalid arguments, func: %s, line: %d, notify_data: %p, user_data: %p\n",
                  __func__, __LINE__, notify_data, user_data);
        return;
    }
    if(!udrv_dev_is_ready(notification_data->dev)){                                      // if dev followed not ready
        dev_info(dev, "my following device '%s' hotplug state was changed(NOT ready), I will be OFFLINE:)\n",
                 notification_data->dev->name);
        udrv_dev_set_state(dev, UDRV_DEVICE_STATE_OFFLINE);                              // set subscriber offline
    }
    else{                                                                                // if publisher state changed
        dev_info(dev, "my following device '%s' hotplug state was changed(Ready), I will be ONLINE:)\n",
                 notification_data->dev->name);
        udrv_dev_clr_state(dev, UDRV_DEVICE_STATE_OFFLINE);                              // set subscriber online
    }
    dev_info(dev, "device '%s' state: %d, device '%s' state: %d\n", notification_data->dev->name,
             udrv_dev_get_state(notification_data->dev), dev->name, udrv_dev_get_state(dev));
}
```

13.3: if the publisher state not in ready state, the subscriber drop all cached data related to it.

```blurtext
/* udrv/drivers/driver_rip.c */
static void rip_drop_cache_if_need(struct device* dev){
    struct rip_devdata* devdata;
    if(dev){
        devdata = udrv_dev_get_devdata(dev);                                 // get devdata of the subscriber dev
        if(devdata && devdata->cache_enabled){                               // if devdata exist & cache enabled
            udrv_ll_cache_group_clear_all(devdata->group);
            dev_info(dev, "cache data has been removed!");
        }
    }
}
```

<p style="margin-bottom: 20px;"></p>

14 udrv.json: a config case for ntio-rip device.

```blurtext
{
    "configs": {...},
    "file": {
        "compatible": "udrv file controller",            // controller driver of ntio rip dev
        "ntio-rip": {                                    // ntio-rip device
            "compatible": "udrv rip device",             // dev->drv is rip device driver
            "following": "ntio",                         // it's following the ntio device
            "cache-enabled": true,                       // enable cache
            "path": "vroot/rip"                          // sysfs uobj attr path (created when init board fs???)
        },
    },
    "misc": {
        "compatible": "udrv misc controller",
        "ntio": {                                        // ntio device (the dev that ntio-rip is following)
            "compatible": "udrv hotplug device",         // dev->drv = hotplug dev drv
            "events": {
                "hotplug": {                             // when plugin/out, cpld set reg & report irq to cpu
                    "intrdev": "intr_ntio_presence",     // irq dev to indicate ntio-rip hotplug event
                    "reg": "cpld.ntio_presence",         // cpld field to show ntio-rip plug state
                    "privatedata": "0x1"                 // if cpld.ntio_presence == 0x1, the state is plugin
                }
            }
        },
    },
}
```

notes from the above udrv rip device instance configuration:  
1 the rip device driver's controller device driver is udrv file controller driver, thus each ntio-rip
device read & write operation is actually invoking the dev's parent controller's registed operation,
with ref to 3.1 rip_read_write section and blog post linux-udrv-file-controller-drv-codewalk.  
2 "following": "ntio" means that current ntio-rip device instance subscribes the ntio device hotplug state
(offline or online), see "ntio" device configuration for example, and with reference to 13.2
udrv_ll_dev_device_following_default_following_device_state_changed_cb definitions.
