---
layout: post
title: "linux device model (linux, device)"
author: "melon"
date: 2024-06-20 23:21
categories: "2024"
tags:
  - linux
---

device driver are the glue between the os and io devices.
device model represents the actual connections between buses and the devices they control.

<hr>

### # the design inspiration of device model
given the following design architecture, for a nic dev named as nic_x connected with cpu
by the internal bus (addr bus, data bus, control bus, irq pins\...):

```txt

                    ┌───────────────────────────────────────────────────┐
                    │                      soc                          │
                    │     ┌───────┐                   ┌───────────┐     │
                    │     │       │    address bus    │           │     │
                    │     │       │ ----------------> │           │     │
                    │     │       │                   │           │     │
                    │     │       │                   │           │     │
                    │     │       │     data bus      │           │     │
                    │     │  cpu  │ <---------------> │   nic_x   │     │
                    │     │       │                   │ controler │     │
                    │     │       │                   │           │     │
                    │     │       │    control bus    │           │     │
                    │     │       │ <---------------> │           │     │
                    │     │       │                   │           │     │
                    │     └───────┘                   └───────────┘     │
                    │                                                   │
                    └───────────────────────────────────────────────────┘

```

1 maintain device-specific info inside device drivers  
it's intuitive to store cpu related info like base addr, irq num\... inside drv of nic_x:

```text
#define NICX_BASE 0x0001   // base addr
#define NICX_INTERRUPT 2   // irq num

int nicx_send(){
    writel(NICX_BASE + REG, 1);
    ...
}

int nicx_init(){
    request_init(NICX_INTERRUPT, ...);
    ...
}
```

while, the above mappings the drv and dev in a 1-to-1 pattern.
so maintaining one drivers for each device is hard due to there are tons of devices
existed and incoming.

here comes the need to design a device driver model to fit for adaptions to kinds of device.
basically, the driver wont change much cross cpu types, and should be platform-independant.
thus, the board-specific info (utilized by cpu) should not appear in driver model:
i.e. base addr, irq num.

<p style="margin-bottom: 20px;"></p>

2 separate board-specific info with driver  
a method to segregate the info out of drv is: ask the dev, what's your base addr, irq num?

```txt

                                     ┌────────────────┐
                                     │                │
                                     │  nic_x driver  │
                                     │                │
                          irq num?   └────┬──┬──┬─────┘ irq num?
                          base addr?      |  |  |       base addr?
                         ┌----------------┘  |  └----------------┐
                         |         base addr?|irq num?           |
                 ┌───────+───────┐   ┌───────+───────┐   ┌───────+───────┐
                 │               │   │               │   │               │
                 │ nic_x board A │   │ nic_x board B │   │ nic_x board C │
                 │               │   │               │   │               │
                 └───────────────┘   └───────────────┘   └───────────────┘

```

however, this method still has coupled part between driver and devices: the ask action
simply require maintain some device specific callbacks in the driver implementation.

<p style="margin-bottom: 20px;"></p>

3 setup an adapter between driver and devices  
the adapter layer could assist us in making adaptions to different board-level informations.
thus, the driver could get the hardware info through relative static & stable adaptor api.

```txt

                                    ┌────────────────┐
                                    │                │
                                    │  nic_x driver  │
                                    │                │
                                    └─────┬──┬──┬────┘
                                          |  |  |
                  ┌───────────────────────┴──┴──┴───────────────────────┐
                  │                         bus                         │
                  └───────────────────────┬──┬──┬───────────────────────┘
                          irq num?        |  |  |       irq num?
                          base addr?      |  |  |       base addr?
                         ┌----------------┘  |  └----------------┐
                         |         base addr?|irq num?           |
                 ┌───────+───────┐   ┌───────+───────┐   ┌───────+───────┐
                 │               │   │               │   │               │
                 │ nic_x board A │   │ nic_x board B │   │ nic_x board C │
                 │               │   │               │   │               │
                 └───────────────┘   └───────────────┘   └───────────────┘
                arch/arm/board-A.c  arch/arm/board-B.c   arch/arm/board-C.c
```

based on the above scheme, linux device model is separated into 3 parts: driver, bus, device.

```txt
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 subject │ functionalities                                                              │ code
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 device  │ base addr, irq num, clock, dma, reset...                                     │ arch/...
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 driver  │ implement the function of the device: send/recv pkts, read/write sd card...  │ drivers/net drivers/mmc ...
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 bus     | complete the association between device and drivers                          │ drivers/plt drivers/pci ...
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

<p style="margin-bottom: 20px;"></p>

4 conclusion  
fill hardware info into device and register device to bus, then the device info is known to bus.
register the device driver into the bus, then the driver capabilities are know to bus.
finally the bus could buildup the relationship of the device & driver by the match method:

```text
static int platform_match(struct device* dev, struct device_driver* drv){
    struct platform_device* pdev = to_platform_device(dev);
    struct platform_driver* pdrv = to_platform_driver(drv);

    if(pdev->driver_override){                                    // when override set, only bind to matched drv
        return !strcmp(pdev->driver_override, drv->name);
    }
    if(of_driver_match_device(dev, drv)){                         // attempt OF style match first
        return 1;
    }
    if(acpi_driver_match_device(dev, drv)){                       // then try acpi style match
        return 1;
    }
    if(pdrv->id_table){                                           // then try id table match
        return platform_match_id(pdrv->id_table, pdev) != NULL;
    }
    return (strcmp(pdev->name, drv->name) == 0);                  // finally fallback to name match
}
```

in this way the decoupling of device & driver is done.

<p style="margin-bottom: 20px;"></p>

5 improvement of device model: dts  
consider the driver-bus-device.c model above, each time new feature released on the device,
the lowest-level board code xxx.c need to be changed for adaptions.
as more and more devices are supported in linux kernel, device maintainence is getting harder.
for more details, please ref to dts blog article.

<hr>

### # buses
a bus is a channel between the processor and one or more devices.
for the purposes of the device model, all devices are connected via a bus,
even if it is an internal, virtual, platform bus.
buses can plug into each other, i.e usb controller is usually a pci device.

bus is represented by the bus_type structure defined in linux/device/bus.h:

```text
struct bus_type {                                                     // bus info of a device
    const char*                     name;                             // name of bus
    const char*                     dev_name;                         // for subsys to enum dev: ("foo%u", dev->id)
    struct device*                  dev_root;                         // default parent for dev
    const struct attribute_group**  bus_groups;                       // default attr of the bus
    const struct attribute_group**  dev_groups;                       // default attr of dev on bus
    const struct attribute_group**  drv_groups;                       // default attr of dev drv on bus

    int  (*match)(struct device* dev, struct device_driver* drv);     // called >=1 when dev/drv add on bus
    int  (*uevent)(struct device* dev, struct kobj_uevent_env* env);  // called when dev add/rm/other (add dev env var)
    int  (*probe)(struct device* dev);           // called when new dev add to bus, callback dev's probe for dev init
    void (*sync_state)(struct device* dev);                   // called to sync dev state to sw state
    int  (*remove)(struct device* dev);                       // called when rm dev from bus
    void (*shutdown)(struct device* dev);                     // called at shutdown to quiesce the dev
    int  (*online)(struct device* dev);                       // called to put dev back online (after offline it)
    int  (*offline)(struct device* dev);                      // called to put dev offline for hot-removal
    int  (*suspend)(struct device* dev, pm_message_t state);  // called when dev try goto sleep mode
    int  (*resume)(struct device* dev);                       // called to bring its dev out of sleep mode
    int  (*num_vf)(struct device* dev);                       // called to find the num of v-func the dev support
    int  (*dma_configure)(struct device* dev);                // called to setup dma config for dev on bus

    const struct dev_pm_ops*  pm;                // power management op of bus, callback drv pm-ops
    const struct iommu_ops*   iommu_ops;         // attach iommu drv op for the bus, for drv bus-specific setup
    struct subsys_private*    p;                 // priv data of drv code (only drv core can touch)
    struct lock_class_key     lock_key;          // for lock validator
    bool                      need_parent_lock;  // whether lock the dev's par when probe or rm dev on this bus
};
```

<p style="margin-bottom: 20px;"></p>

1 bus registration  
the example below includes a virtual bus implementation called lddbus.
this bus sets up its bus_type structure as follows:

```text
struct bus_type ldd_bus_type = {
    .name = "ldd",
    .match = ldd_match,
    .hotplug  = ldd_hotplug,
};
```

note: very few of the bus_type fields require initialization, most of that is handled
by the device model core.
however, the bus name & the method along with must be specified.

inevitably, a new bus must be registered with the system via a call to bus_register.
the lddbus code does so in this way:

```text
ret = bus_register(&ldd_bus_type);
if(ret){
    return ret;
}
```

the register call can fail, thus the return value must always be checked.
if succeeds, the new bus subsystem has been added to the system, which is visible under
/sys/bus and available for adding devices.

sometimes it's necessary to remove a bus from the system when associated module got removed:

```text
void bus_unregister(struct bus_type *bus);
```

<p style="margin-bottom: 20px;"></p>

2 bus methods  
there are several methods defined for bus_type structure, to allow bus code as intermediary
between the device core and individual drivers.
the following list some methods defined in the 2.6.10 kernel:

2.1 match  
match method is called whenever new device or driver added for this bus, and
will return nonzero value if the given dev can be handled by the given drv.

```text
int (*match)(struct device* device, struct device_driver* driver);
```

kernel core is not responsible for matching dev and drv for all possible bus type,
so the match must be handled at bus level.
in match function for real hardware, there are some comparisons between the device id and driver id.

e.g. lddbus driver has very simple match function, simply compareing the name of the drv and dev:

```text
static int ldd_match(struct device* dev, struct device_driver* driver){
    return !strncmp(dev->bus_id, driver->name, strlen(driver->name));
}
```

2.2 hotplug  
the method below allow bus to add environment variables prior to generate a hotplug event in userspace.

```text
int (*hotplug) (struct device* device, char** envp, int num_envp, char* buffer, int buffer_size);
```

e.g. lddbus hotplug method looks like this:

```text
static int ldd_hotplug(struct device* dev, char** envp, int num_envp, char* buffer, int buffer_size){
    envp[0] = buffer;
    if(snprintf(buffer, buffer_size, "LDDBUS_VERSION=%s", Version) >= buffer_size){
        return -ENOMEM;
    }
    envp[1] = NULL;
    return 0;
}
```

2.3 iterate over dev and drv  
when writing bus-level code, there's need to perform op on all dev or drv
registered with certain bus.
one way is to dig into the struct bus_type, but it is better to use the build-in helper func.

1) to operate on every dev known to the bus:

```text
int bus_for_each_dev(struct bus_type* bus, struct device* start, void* data, int (*fn)(struct device*, void*));
```

the above func iter over every dev on bus, passing the associated struct dev & data to fn,
if start is NULL, then iter begins with the first dev on bus,
else iter starts with the first dev after start.
if fn ret a nonzero value, iter stops and return the val.

2) to operate on every drv known to the bus:

```text
int bus_for_each_drv(struct bus_type* bus, struct device_driver* start, void* data,
                     int (*fn)(struct device_driver*, void*));
```

the mechanism is just the same except for it iter for drvs.

note that both of the above two func hold the bus subsystem's reader/writer
semaphore for the duration of their work. the attempt to use them with context interleaved
will deadlock, each will be trying to obtain the same semaphore:

```text
bus_for_each_dev(...){                      // in ctx of for_each_dev: try possess the r/w sema
    buf_for_each_drv(...);                  // invoke for_each_drv: try get the same r/w sema -> blocked
}                                           // cannot release the sema -> deallock
```

moreover, operations to modify the bus int the context of the above 2 iter funcs
(e.g. unregister dev) will also lock up. so be cautious to use bus_for_each func.

2.4 bus attributes  
the bus layer provide an itf for addition attr like other layer in linux dev model does.
bus_attribute type defined in linux/device/bus.h as follows:

```text
struct bus_attribute {
    struct attribute attr;
    ssize_t (*show)(struct bus_type* bus, char* buf);                       // display attr
    ssize_t (*store)(struct bus_type* bus, const char* buf, size_t count);  // set attr
};
```

a helper macro is provided as follows for compile-time create and init of bus_attribute:

```text
BUS_ATTR(name, mode, show, store);
```

any attr belonging to the bus should be created/removed explicitly by following api:

```text
int  bus_create_file(struct bus_type* bus, struct bus_attribute* attr);     // create attr
void bus_remove_file(struct bus_type* bus, struct bus_attribute* attr);     // remove attr
```

e.g. lddbus drv creates a simple attr file containing the src version num.
the show method and struct bus_attribute are set up as follows:

```text
static ssize_t show_bus_version(struct bus_type* bus, char* buf){
    return snprintf(buf, PAGE_SIZE, "%s\n", Version);
}

static BUS_ATTR(version, S_IRUGO, show_bus_version, NULL);
```

create the attr file is done at module load time:

```text
if(bus_create_file(&ldd_bus_type, &bus_attr_version)){
    printk(KERN_NOTICE "Unable to create version attribute\n");
}
```

which will create an attr file at /sys/bus/ldd/version containing the rev for lddbus code.

<hr>

### # devices
at lowest level, each dev in linux is represented by an instance of struct device:

```text
struct device {
    struct device* parent;                    // parent dev: can be a bus or a controller
    struct device_private* p;                 // private data of driver core of the dev
    struct kobject kobj;                      // abstract base
    const char* init_name;                    // dev name
    const struct device_type* type;           // dev type with type-specific info
    struct mutex mutex;                       // mutex for sync calls to dev
    struct bus_type* bus;                     // type of bus the dev on
    struct device_driver* driver;             // the driver to allocate dev instance
    void* platform_data;                      // plt data for the dev
    void* driver_data;                        // drv specific info
    struct dev_pm_info power;                 // power mgnt, ref: Documentation/power/devices.txt
    struct dev_pm_domain* pm_domain;          // callbacks for power domain changes
#ifdef CONFIG_GENERIC_MSI_IRQ_DOMAIN
    struct irq_domain* msi_domain;            // msi domain for the dev
#endif
#ifdef CONFIG_PINCTRL
    struct dev_pin_info* pins;                // pin mgnt, ref: Documentation/pinctrl.txt
#endif
#ifdef CONFIG_GENERIC_MSI_IRQ
    struct list_head msi_list;                // host msi descriptor
#endif
#ifdef CONFIG_NUMA
    int numa_node;                            // numa node the dev close to
#endif
    u64* dma_mask;                            // mask for dma'ble dev
    u64 coherent_dma_mask;                    // coherent mask
    unsigned long dma_pfn_offset;             // dma mem range offset relative to ram
    struct device_dma_parameters* dma_parms;  // ll drv code set this to notice iommu of segment limitations
    struct list_head dma_pools;               // dma pools
    struct dma_coherent_mem* dma_mem;         // for dma mem override
#ifdef CONFIG_DMA_CMA
    struct cma* cma_area;                     // contiguous mem area for dma alloc
#endif
    struct dev_archdata archdata;             // arch-specific data
    struct device_node* of_node;              // associated dev tree node
    struct fwnode_handle* fwnode;             // associated dev node of firmware
    dev_t devt;                               // for sysfs /dev
    u32 id;                                   // dev id
    spinlock_t devres_lock;                   // spinlock to protect dev resource
    struct list_head devres_head;             // reource list of the dev
    struct klist_node knode_class;            // classlist node of the dev
    struct class* class;                      // class of the dev
    const struct attribute_group** groups;    // attr groups (optional)
    void (*release) (struct device* dev);     // callback to release dev after reference is 0 (i.e. set by bus drviver)
    struct iommu_group* iommu_group;          // iommu group of the dev
    bool offline_disabled:1;                  // if set, dev will permanently online
    bool offline:1;                           // set after invocation of bus_type offline done
};
```

<p style="margin-bottom: 20px;"></p>

1 device registration  
the register & unregister api:
```text
int  device_register(struct device* dev);
void device_unregister(struct device* dev);
```

e.g. bus is also a device and must be registered separately.
for simplicity, the lddbus module supports only a single virtual bus,
so the lddbus device can be setup by its drv at compile time like:

```text
static void ldd_bus_release(struct device* dev){
    printk(KERN_DEBUG "lddbus release\n");
}

struct device ldd_bus = {
    .bus_id   = "ldd0",
    .release  = ldd_bus_release
};
```

this is a top-level bus, so the parent and bus fields are left NULL.
it's a simple bus named as ldd0.  
device lddbus is registered with:

```text
ret = device_register(&ldd_bus);
if(ret){
    printk(KERN_NOTICE "Unable to register ldd0\n");
}
```

once that call is complete, the lddbus is visible under /sys/devices in sysfs.
any devices added to this bus will show up under /sys/devices/ldd0/.

<p style="margin-bottom: 20px;"></p>

2 device attributes  
device entries in sysfs can have attributes. the related dev attr structure is as:

```text
struct device_attribute {
    struct attribute attr;
    ssize_t (*show)(struct device* dev, char* buf);
    ssize_t (*store)(struct device* dev, const char* buf, size_t count);
};
```

the attr struct can be set up at compile time by macro as:

```text
DEVICE_ATTR(name, mode, show, store);
```

the attribute files can be handled with following api:

```text
int  device_create_file(struct device* device, struct device_attribute* entry);
void device_remove_file(struct device* dev, struct device_attribute* attr);
```

the dev_attrs field in bus_type struct is the default attr for every dev added to
that bus.

<p style="margin-bottom: 20px;"></p>

3 device structure embedding  
struct device contains the info for device model core to model the system.
however, most subsystems requires additional info of the dev hosted.
thus, it's rare for devices to be represented as bare struct device, typicaly a higher-level
wrapper representation of struct device it utilized.
i.e. the struct pci_dev or struct usb_device maintain a struct device inside.

e.g. lddbus drv creates its own device type: ldd_device and use the type for dev registration:

```text
struct ldd_device {
    char* name;
    struct ldd_driver* driver;
    struct device dev;
};

#define to_ldd_device(dev) container_of(dev, struct ldd_device, dev);
```

real devices contain info like: the vendor, device model, device configuration,
resources used, and so on. a detailed examples can be found at struct pci_dev
at linux/pci.h or struct usb_device at linux/usb.h.

the helper macro is used to turn ptr to the embedded dev structure into ldd_device ptr easily.

the lddbus registration interface is as:

```text
int register_ldd_device(struct ldd_device* ldddev){
    ldddev->dev.bus = &ldd_bus_type;
    ldddev->dev.parent = &ldd_bus;
    ldddev->dev.release = ldd_dev_release;
    strncpy(ldddev->dev.bus_id, ldddev->name, BUS_ID_SIZE);
    return device_register(&ldddev->dev);
}

EXPORT_SYMBOL(register_ldd_device);
```

<hr>

### # device drivers
device model is able to track all drv known to the system, which enable drv core to match
up drv with new coming devs.
device drivers can export info and config variables that are independent of
any specific device, for example.

drv are defined as:

```text
struct device_driver {
    const char* name;                                        // device driver name
    struct bus_type* bus;                                    // which bus is the device driver belong to
    struct module* owner;                                    // module owner
    const char* mod_name;                                    // used for build-in modules
    bool suppress_bind_attrs;                                // disable bind/unbind via sysfs
    enum probe_type probe_type;                              // sync or async
    const struct of_device_id* of_match_table;               // open firmware table
    const struct acpi_device_id* acpi_match_table;
    int (*probe)(struct device* dev);                        // called to query existence of dev, drv dev match or not
    int (*remove)(struct device* dev);                       // called when dev is removed, unbind dev from drv
    void (*shutdown)(struct device* dev);                    // called when shutdown to quiesce the dev
    int (*suspend)(struct device* dev, pm_message_t state);  // called to put dev to sleep (low power)
    int (*resume)(struct device* dev);                       // called to bring dev back from sleep
    const struct attribute_group** groups;                   // default attr created by drv core
    const struct dev_pm_ops* pm;                             // power mgnt op of dev matched with cur drv
    struct driver_private* p;                                // private data of cur drv core
};
```

the register & unregister func are as:

```text
int  driver_register(struct device_driver* drv);
void driver_unregister(struct device_driver* drv);
```

the attribute structure is as:

```text
struct driver_attribute {
    struct attribute attr;
    ssize_t (*show)(struct device_driver* drv, char* buf);
    ssize_t (*store)(struct device_driver* drv, const char* buf, size_t count);
};

DRIVER_ATTR(name, mode, show, store);
```

the attr files can be created by:

```text
int driver_create_file(struct device_driver* drv, struct driver_attribute* attr);
void driver_remove_file(struct device_driver* drv, struct driver_attribute* attr);
```

the struct bus_type contains a field (drv_attrs) that points to a set of default attributes,
which are used for all drv associated with that bus.

<p style="margin-bottom: 20px;"></p>

1 driver structure embedding  
the struct device_driver is usually embedded in a higher-level, bus-specific struct.  
e.g. lddbus subsystem defined its own ldd_driver structure as:

```text
struct ldd_driver {
    char* version;
    struct module* module;
    struct device_driver driver;
    struct driver_attribute version_attr;
};

#define to_ldd_driver(drv) container_of(drv, struct ldd_driver, driver);
```

bus-specific driver registration function is:

```text
int register_ldd_driver(struct ldd_driver* driver){
    int ret;
    driver->driver.bus = &ldd_bus_type;
    ret = driver_register(&driver->driver);            // register ll device_driver to the core
    if(ret){
        return ret;
    }
    driver->version_attr.attr.name = "version";        // runtime setup lddbus version info, the DRIVER_ATTR wont work
    driver->version_attr.attr.owner = driver->module;
    driver->version_attr.attr.mode = S_IRUGO;
    driver->version_attr.show = show_version;
    driver->version_attr.store = NULL;
    return driver_create_file(&driver->driver, &driver->version_attr);
}
```

lddbus ver info is exposed for every drv instance of ldd_driver type.

owner of the attribute is set to the ldd drv module rather than the ldd bus module,
although the function to implement the version attr is defined in lddbus.  
check the implementation of show function as an evidance:

```text
static ssize_t show_version(struct device_driver* driver, char* buf){
    struct ldd_driver* ldriver = to_ldd_driver(driver);
    sprintf(buf, "%s\n", ldriver->version);              // ldd drv own the version attr
    return strlen(buf);
}
```

if version attr is owned by lddbus, when bus go away while a userspace proc tried to read the
version number, the result will go messy.

designate the drv module as the owner of the attr prevents the bus module getting unloaded,
when userspace proc holds the attr file open.

<hr>

### # reference
ref: https://www.linuxtv.org/downloads/v4l-dvb-internals/device-drivers/devdrivers.html#id-1.4.2  
ref: http://www.makelinux.net/ldd3/chp-14-sect-4.shtml
