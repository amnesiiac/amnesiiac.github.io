---
layout: post
title: "sysfs intro (linux, sysfs, kernel)"
author: "melon"
date: 2023-12-15 21:08
categories: "2023"
tags:
  - linux
  - kernel
---

sysfs provides a dynamic and intuitive interface to kernel data structures.

sysfs is a ram-based virtual filesystem that provides a way to access and manipulate information about
kernel data structures.
sysfs allow for both the view of device attributes and the modification of certain parameters,
facilitating dynamic interaction between userspace and the kernel, which makes it a valuable tool for
system management and diagnostics.

the sysfs can also be seen as a layout of the kobjects, with each obj corresponding to one dir under sysfs.
each kobject has attributes like parent dir (parent dir in sysfs), name (sysfs dir name), type (struct kobj_type),
kref containing reference count, kset for grouping related kobjects together\...

the below provide a basic definition of kobject struct:

```text
struct kobject {
    const char*              name;
    struct list_head*        ntry;
    struct kobject*          parent;
    struct kset*             kset;
    const struct kobj_type*  ktype;                        // see 1
    struct kernfs_node*      sd;                           // sysfs directory entry
    struct kref              kref;

    unsigned int             state_initialized:1;
    unsigned int             state_in_sysfs:1;
    unsigned int             state_add_uevent_sent:1;
    unsigned int             state_remove_uevent_sent:1;
    unsigned int             uevent_suppress:1;

#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
    struct delayed_work      release;
#endif
};
```

1 the kobj_type defines the behavior of the kobject, containing: 1) a list of attributes as sysfs files for
user-interaction, and 2) the sysfs_ops operation structure to define the store & show callbacks.

```text
struct kobj_type {
    void (*release)(struct kobject* kobj);                 // called when kobj is released by kobject_put
    const struct sysfs_ops*        sysfs_ops;              // see 2
    const struct attribute_group** default_groups;         // see 3
    const struct kobj_ns_type_operations* (*child_ns_type)(const struct kobject* kobj);
    const void* (*namespace)(const struct kobject* kobj);
    void (*get_ownership)(const struct kobject* kobj, kuid_t* uid, kgid_t* gid);
};
```

2 the kobject->ktype->sysfs_ops structure defined the store show callback for each kobj, all supported attributes
files can share the callbacks to response to the interactive operations: cat, echo\...

```text
struct sysfs_ops {
    ssize_t (*show)(struct kobject*, struct attribute*, char*);
    ssize_t (*store)(struct kobject*, struct attribute*, const char*, size_t);
};
```

3 attributes is used to expose and manipulate kernel information for userspace.
attr take the form of regular files, each with a unique name (the file name), where you could bind a dedicated
file operations structure (see below).

```text
/* linux/include/linux/sysfs.h */
struct attribute_group {
    const char*              name;
    umode_t (*is_visible)(struct kobject*, struct attribute*, int);
    umode_t (*is_bin_visible)(struct kobject*, struct bin_attribute*, int);
    struct attribute**       attrs;          // pointer to list of attributes (NULL terminated).
    struct bin_attribute**   bin_attrs;      // pointer to list of binary attributes (NULL terminated).
};

struct attribute {
    const char*              name;           // unique name (to bind dedicated operation)
    umode_t                  mode;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    bool                     ignore_lockdep:1;
    struct lock_class_key*   key;
    struct lock_class_key    skey;
#endif
};
```

when a show (read) or store (write) operation is performed on a sysfs file, the sysfs_ops methods defined in
kobj_type of your kobject are always invoked.

4 container_of: cast a pointer to the member of a struct to the pointer of its containing struct.
by using container_of on the struct attribute passed to the sysfs_ops methods, one can get pointer to
the kobj_attribute struct and invoke the more specific show and store operations for their single attribute obj.

```text
/* linux/tools/include/linux/kernel.h */
// @ptr:	the ptr to the member
// @type:	the type of the container struct of the ptr->member embedded in
// @member:	the name of the ptr->member

#ifndef container_of
#define container_of(ptr, type, member) ({                 \
    const typeof(((type*)0)->member)* __mptr = (ptr);      \
    (type *)((char *)__mptr - offsetof(type, member)); })
#endif
```

with ref to the naming convention of the kernel, take __mptr as case, the __ means it's an internal variable,
the m denotes it's an inter-mediate variable.

<hr>

### # sysfs kernel module example (todo)
try understand & comment on this examples, understand the kernel interface for the basic kdrv,
extract the common & main trunk of these 3 look-the-same ko.

0 makefile: make helkper script for the following 3 examples.

```text
kobject_example_obj-objs := kobject_example.o
kobject_example_kobj-objs := kobj_attribute_example.o
kobject_example_kset-objs := sysfs_kset_example.o

obj-m := kobject_example_obj.o kobject_example_kobj.o kobject_example_kset.o

COMPILE_DIR=$(PWD)
KDIR = /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(COMPILE_DIR) modules

clean:
	rm -f -v *.o *.ko
```

<p style="margin-bottom: 20px;"></p>

1 kobject_example.c: introduction to kobject basics  
this module shows how to create a simple subdirectory in sysfs called /sys/kernel/kobject-example,
in which 2 files are created: my_int and my_second_int.
once an integer got written to the files created, then it can be read back from them later.

```text
#include <linux/kobject.h>
#include <linux/string.h>
#include <linux/sysfs.h>
#include <linux/module.h>
#include <linux/init.h>

struct kobject example_kobj;                               // ...
static int my_int = 0;
static int my_second_int = 0;

// file read op: write to tmp buf from the kobj (hold data before passing to userspace)
static ssize_t my_file_show(struct kobject* kobj, struct attribute* attr, char* buf){
    int ret = 0;
    if(strncmp(attr->name, "my_int", 6)){                  // attr->name: name of the attribute for read
        ret = sysfs_emit(buf, "%d\n", my_int);             // write from sysfs to tmp buf (page_size aware)
    }
    else{
        ret = sysfs_emit(buf, "%d\n", my_second_int);
    }
    return ret;
}

// file write op: from buf (filled with content from userspace by sys_write) to sysfs attr file
static ssize_t my_file_store(struct kobject* kobj, struct attribute* attr, const char* buf, size_t count){
    int ret;
    if(strncmp(attr->name, "my_int", 6)){                  // attr->name: write to the attr->name (sysfs filename)
        ret = kstrtoint(buf, 10, &my_int);                 // convert str to int (from buf to &my_int)
    }
    else{
        ret = kstrtoint(buf, 10, &my_second_int);
    }
    if(ret < 0){
        return ret;
    }
    return count;
}

static void my_file_release(struct kobject* kobj){         // called in kobject_put to destory the kobj
    printk("anything to do!\n");                           // here nothing need to do
}

struct sysfs_ops my_sysfs_ops = {                          // combine the sysfs operations: r/w
    .show = my_file_show,
    .store = my_file_store,
};

static struct attribute my_file_attribute = {              // define attr for the sysfs: my_int
    .name = "my_int",                                      // the regular file name
    .mode = 0664,
};

static struct attribute my_second_file_attribute = {       // define attr for the sysfs: my_second_int
    .name = "my_second_int",                               // the regular file name
    .mode = 0664,
};

static struct attribute* my_file_attrs[] = {               // buildup attr arr to bind to the kobj
    &my_file_attribute,
    &my_second_file_attribute,
    NULL,
};
                                                           // equivalent form:
ATTRIBUTE_GROUPS(my_file);                                 // struct attribute_group the my_file_groups = {
                                                           //     .attrs = my_file_attrs,
                                                           // };

static const struct kobj_type my_ktype = {                 // define custom ktype for kobj above
    .sysfs_ops = &my_sysfs_ops,                            // set sysfs_ops
    .release = my_file_release,                            // set kobj release callback
    .default_groups = my_file_groups,                      // set of attr created when kobj of ktype is registered
};

static int __init example_init(void){                      // module init: create kobj
    int retval;
    retval = kobject_init_and_add(                                       // type as ktype, name as kobject_example
        &example_kobj, &my_ktype, kernel_kobj, "%s", "kobject_example"   // path: /sys/kernel/kobject_example
    );
    if(retval){
        return -ENOMEM;
    }
    return 0;
}

static void __exit example_exit(void){                     // module exit: call kobject_put -> release
    kobject_put(&example_kobj);
}

module_init(example_init);
module_exit(example_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("melon <melon@melon.com>");
```

test the above sample code (todo):

```text
$ cd /sys/kernel/kobject_example
$ echo 2 > my_int
$ echo 3 > my_second_int
$ cat my_int
$ 2
$ cat my_second_int
$ 3
```

<p style="margin-bottom: 20px;"></p>

2 kobj_attribute_example.c: further improve the kobject attributes  
this module shows how to create a simple subdirectory in sysfs called /sys/kernel/kobject-example.
in that directory, 2 files are created: "my_int" and "my_second_int".
if an integer is written to these files, it can be later read out of it.

```text
#include <linux/kobject.h>
#include <linux/string.h>
#include <linux/sysfs.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/slab.h>

struct my_example_data {                                   // define wrapper type
    struct kobject example_kobj;                           // member 1: kobj
    int            my_int;                                 // member 2: int
    char           my_buffer[256];                         // member 3: my_buffer
};

// file read op: write from the kobj to tmp buf (allocated k page, pending to pass to userspace)
static ssize_t my_file_show(struct kobject* kobj, struct kobj_attribute* attr, char* buf){
    struct my_example_data* my_data;
    my_data = container_of(kobj, struct my_example_data, example_kobj);  // get the wrapper instance of kobj
    return sysfs_emit(buf, "%d\n", my_data->my_int);       // care the dimension of userspace buf for output
}

// file write op: write from the userspace buf (filled by content from sys_write) to the sysfs kobj attr
static ssize_t my_file_store(struct kobject* kobj, struct kobj_attribute* attr, const char* buf, size_t count){
    struct my_example_data* my_data;
    my_data = container_of(kobj, struct my_example_data, example_kobj);
    if(kstrtoint(buf, 10, &my_data->my_int) < 0){          // str to int (copy from buf to my_data->my_int)
        return 0;
    }
    return count;
}

// buffer read op: write from my_buffer to tmp buf (allocated k page, which will be mapped in userspace)
static ssize_t my_buffer_show(struct kobject* kobj, struct kobj_attribute* attr, char* buf){
    struct my_example_data* my_data;
    my_data = container_of(kobj, struct my_example_data, example_kobj);  // get the wrapper instance of kobj
    return sysfs_emit(buf, "%s\n", my_data->my_buffer);
}

// buffer write op: write from the userspace buf (filled by content from sys_write) to the sysfs kobj attr
static ssize_t my_buffer_store(struct kobject* kobj, struct kobj_attribute* attr, const char* buf, size_t count){
    struct my_example_data* my_data;
    my_data = container_of(kobj, struct my_example_data, example_kobj);  // get the wrapper instance of kobj
    strncpy(my_data->my_buffer, buf, 256);                 // copy from buf (k page mapped to userspace) to my_buffer
    return count;
}

static void my_file_release(struct kobject* kobj){
    printk("anything to do!\n");                           // kobject_put -> release to destory the kobj
}

// defines my_int & my_buffer attr under /sys/kernel/kobjec-example-2/
static struct kobj_attribute my_file_attribute = __ATTR(my_int, 0664, my_file_show, my_file_store);
static struct kobj_attribute my_buffer_attribute = __ATTR(my_buffer, 0664, my_buffer_show, my_buffer_store);

static struct attribute* my_file_attrs[] = {               // the attributes array to bind to the kobject
    &my_file_attribute.attr,
    &my_buffer_attribute.attr,
    NULL,
};
                                                           // equivalent form:
ATTRIBUTE_GROUPS(my_file);                                 // struct attribute_group the my_file_groups = {
                                                           //     .attrs = my_file_attrs,
                                                           // };

static const struct kobj_type my_ktype = {                 // define custom ktype for kobj above
    .sysfs_ops = &kobj_sysfs_ops,                          // set sysfs_ops
    .release = my_file_release,                            // set kobj release callback
    .default_groups = my_file_groups,                      // set of attr created when kobj of ktype is registered
};

struct my_example_data* data = NULL;

static int __init example_init(void){                      // module load callback
    int retval;
    if((data = kzalloc(sizeof(struct my_example_data), GFP_KERNEL)) == NULL){  // alloc k mem for backup mem of data
        return -ENOMEM;
    }

    // init a kobject struct (named kobject_example) & add it to the sysfs hierarchy
    // the kobject path will be: /sys/kernel/kobject_example
    // kobj: zeroed init kobj; ktype: ptr to ktype of the kobj;
    // parent: ptr to parent of this kobj, parent dir in sys hierarchy.

    retval = kobject_init_and_add(&data->example_kobj, &my_ktype, kernel_kobj, "%s", "kobject_example");

    // sysfs module predefine some basic dir for kobject: /source/include/linux/kobject.h
    // kernel_kobj: contains info & config opts related to the kernel itself.
    // mm_kobj: files and subdirs to provide info about mem alloc, page mgnt, and other mem-related kernel features
    // mm_hypervisor: provide info about the hypervisor under which the cur system is operating on.

    if(retval){
        return -ENOMEM;
    }
    return 0;
}

static void __exit example_exit(void){                     // module unload callback
    // --refcount of kobj, if 0, call kobject_cleanup -> the release callback and rm kobj from sysfs
    // for kobj refcount, please ref: linux/latest/source/include/linux/kref.h
    kobject_put(&data->example_kobj);
    kfree(data);                                           // cleanup mem associated with data
}

module_init(example_init);
module_exit(example_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("melon <melon@melon.com>");
```

test the above module:

```text
$ cd /sys/kernel/kobject_example
$ echo 2 > my_int
$ echo "hello sysfs" > my_buffer
$ cat my_int
2
$ cat my_bufferr
hello sysfs
```

<p style="margin-bottom: 20px;"></p>

3 kset_example.c:  
this module shows how to create a simple subdirectory in sysfs called /sys/kernel/kobject-example.
in that directory, 2 files are created: "my_int" and "my_second_int".
if an integer is written to these files, it can be later read out of it.

we will have our two attr (my_int and my_buffer) on every kobj defined inside our kset.

kset is a collection of kobjs, which provide a way to group related kobj together.
kset can be associated with a specific subsystem or a particular type of kernel objects.
the kset's primary purpose is to provide a way to organize kobj in a hierarchical manner,
similar to how dir organize files in a filesystem.

```text
#include <linux/kobject.h>
#include <linux/string.h>
#include <linux/sysfs.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/slab.h>

struct my_example_data {
    struct kobject example_kobj;
    int            my_int;
    char           my_buffer[256];
};

static ssize_t my_file_show(struct kobject* kobj, struct kobj_attribute* attr, char* buf){
    struct my_example_data* my_data;
    my_data = container_of(kobj, struct my_example_data, example_kobj);
    return sysfs_emit(buf, "%d\n", my_data->my_int);
}

static ssize_t my_file_store(struct kobject* kobj, struct kobj_attribute* attr, const char* buf, size_t count){
    struct my_example_data* my_data;
    my_data = container_of(kobj, struct my_example_data, example_kobj);
    if(kstrtoint(buf, 10, &my_data->my_int) < 0){
        return 0;
    }
    return count;
}

static ssize_t my_buffer_show(struct kobject* kobj, struct kobj_attribute* attr, char* buf){
    struct my_example_data* my_data;
    my_data = container_of(kobj, struct my_example_data, example_kobj);
    return sysfs_emit(buf, "%s\n", my_data->my_buffer);
}

static ssize_t my_buffer_store(struct kobject* kobj, struct kobj_attribute* attr, const char* buf, size_t count){
    struct my_example_data* my_data;
    my_data = container_of(kobj, struct my_example_data, example_kobj);
    strncpy(my_data->my_buffer, buf, 256);
    return count;
}

static void my_file_release(struct kobject* kobj){
    struct my_example_data* my_data;
    my_data = container_of(kobj, struct my_example_data, example_kobj);
    printk("freeing %s\n", kobj->name);
    kfree(my_data);
}

// defines my_int & my_buffer attr under /sys/kernel/kobjec-example-2/
static struct kobj_attribute my_file_attribute = __ATTR(my_int, 0664, my_file_show, my_file_store);
static struct kobj_attribute my_buffer_attribute = __ATTR(my_buffer, 0664, my_buffer_show, my_buffer_store);

static struct attribute* my_file_attrs[] = {
    &my_file_attribute.attr,
    &my_buffer_attribute.attr,
    NULL,
};

ATTRIBUTE_GROUPS(my_file);

static const struct kobj_type my_ktype = {
    .sysfs_ops = &kobj_sysfs_ops,                                 // sysfs callback
    .release = my_file_release,                                   // release callback
    .default_groups = my_file_groups,                             // set of attr of the kobj of this ktype
};

static struct kset*     example_kset = NULL;                      // init kset
struct my_example_data* m_data_1 = NULL;
struct my_example_data* m_data_2 = NULL;
struct my_example_data* m_data_3 = NULL;

static struct my_example_data* create_kobject(const char* name){  // kset create helper
    struct my_example_data* m_data;
    int retval;
    if((m_data = kzalloc(sizeof(struct my_example_data), GFP_KERNEL)) == NULL){
        return ERR_PTR(-ENOMEM);
    }
    m_data->example_kobj.kset = example_kset;                     // set the kset for the kobj
    retval = kobject_init_and_add(&m_data->example_kobj, &my_ktype, NULL, "%s", name);  // kobj creation
    if(retval){
        kfree(m_data);
        return ERR_PTR(-ENOMEM);
    }

    // uevent is a notification msg for kernel -> userspace communication through netlink socket or uevent_helper,
    // to trigger userspace udev (a device manager running as a daemon listening for uevent from kernel) or executable
    // (e.g. /sbin/hotplug) to create or remove dev node, start programs, load or remove drivers...
    // ref: https://elixir.bootlin.com/linux/v3.12.74/source/lib/kobject_uevent.c#L255
    // ref: https://docs.kernel.org/core-api/kobject.html#uevents
    // ref: https://en.wikipedia.org/wiki/Udev#Operation

    kobject_uevent(&m_data->example_kobj, KOBJ_ADD);              // send uevent msg kobj_add

    // ways to specify the helper program to handle the uevent:
    // 1) kconfig: CONFIG_UEVENT_HELPER_PATH can be used to assign the executable path (static)
    // 2) echo ${path_to_uevent_helper} > /sys/kernel/uevent_helper (recommended, usually enabled at early boot)

    return m_data;
}

static int __init example_init(void){                             // module init callback
    int retval;
    example_kset = kset_create_and_add("kset_example", NULL, kernel_kobj);  // init kset under /sys/kernel: kset_example
    if(!example_kset){
        return -ENOMEM;
    }
    if((m_data_1 = create_kobject("objA")) == NULL){              // create objA, set the kset
        goto SETA_ERROR;
    }
    if((m_data_2 = create_kobject("objB")) == NULL){              // create objB, set the kset
        goto SETB_ERROR;
    }
    if((m_data_3 = create_kobject("objC")) == NULL){              // create objC, set the kset
        goto SETC_ERROR;
    }
    return 0;
SETC_ERROR:                                                       // if 3rd kobj init failed, del kobj2, kobj1, kset
    kobject_put(&m_data_2->example_kobj);
    kfree(m_data_2);
SETB_ERROR:                                                       // if 2nd kobj init failed, del kobj1, kset
    kobject_put(&m_data_1->example_kobj);
    kfree(m_data_1);
SETA_ERROR:                                                       // if 1st kobj init failed, del kset
    kset_unregister(example_kset);
    return -EINVAL;
}

static void __exit example_exit(void){                            // module unload callback
    kobject_put(&m_data_1->example_kobj);                         // destroy kobj
    kobject_put(&m_data_2->example_kobj);
    kobject_put(&m_data_3->example_kobj);
    kset_unregister(example_kset);                                // unregister kset
}

module_init(example_init);
module_exit(example_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("melon <melon@melon.com>");
```

test the module above (todo, test this on workmachine & update here):

```text
$ cd /sys/kernel/kset_example/
$ find .  | sed -e 's;[^/]*/;|____;g;s;____|; |;g' #printing directory struct
.
|____objB
| |____my_buffer
| |____my_int
|____objC
| |____my_buffer
| |____my_int
|____objA
| |____my_buffer
| |____my_int
```

<hr>

### # reference
1 some kobject usage examples: https://github.com/torvalds/linux/tree/master/samples/kobject  
2 todo
