---
layout: post
title: "inter-core communication on amp arch II (target) (rust, rpc, amp)"
author: "melon"
date: 1111-08-19 20:40
categories: "2024"
tags:
  - virtualization
  - driver
  - icc
---

the previous part I shows the high-level design graphs of the basic infra involved to buildup the bridge for
inter-core communication.

this article focus on giving an exact description about how the inter-core commnunication mechanism is enabled for
an amp architecture (smp linux + vxworks) on the real hardware board.

<hr>

### # the rpmsg kernel drivers (isam-linux-drivers)

```blurtext
./misc/rpmsg_eth_ifc.c
./misc/rpmsg_platform_driver.c
./misc/rpmsg_misc_dev.c
./misc/rpmsg_interface.c
```

1 the driver code of rpmsg virtio device, at isam-linux-drivers/misc/rpmsg_platform_driver.c:

```blurtext
#define DRIVER_VERSION      "0.0.4"

//  0.0.4   Joeri Barbarien     January 26th 2023
//  ioremap_nocache/devm_ioremap_nocache no longer defined from linux 5.6.0 onwards
//
//  0.0.3   Wu Stan             March 15th 2020
//  Use spin lock to replace mutex, to fix scheduleing wile atomic issue, as it's network xmit context
//
//  0.0.2   Wu Stan             May 9th 2020
//  Update dts parse part, to simplfy the code logic
//
//  0.0.1   Wu Stan             Dec 25th 2018
//  Init version of rpmsg platform driver, this driver would find rpmsg config info from DTS,
//  then register a rpmsg virtio device.

#include <linux/version.h>
#include <linux/err.h>
#include <linux/init.h>
#include <linux/interrupt.h>
#include <linux/module.h>
#include <linux/notifier.h>
#include <linux/kobject.h>
#include <linux/of.h>
#include <linux/platform_device.h>
#include <linux/rpmsg.h>
#include <linux/slab.h>
#include <linux/virtio_config.h>       // *
#include <linux/virtio_ids.h>          // *
#include <linux/virtio_ring.h>         // *
#include <linux/virtio.h>              // *
#include <linux/version.h>

#include "kernel_compat.h"
#include "rpmsg_interface.h"

#define DST_CPU_0              0
#define DST_CPU_1              1
#define LOG_DETAIL             2
#define LOG_BASIC              1
#define LOG_NONE               0

#define VIRTQUEUE_RX           0
#define VIRTQUEUE_TX           1

#define VRING_NUM_PER_VIRTDEV  2
#define VQUEUE_NUM_PER_VIRTDEV 2

struct nokia_rpmsg_vproc {
    struct virtio_device      vdev;                        // ...
    u64                       vring[VRING_NUM_PER_VIRTDEV];
    u64                       data_buffer_pool;

    /* used in virtqueue_notify, it's in network tx context, preemption maybe disable, so should only spinlock here */
    spinlock_t                spin_lock;
    struct virtqueue*         vq[VQUEUE_NUM_PER_VIRTDEV];
    u32                       rpmsg_role;                  // master (1) or slave (2)
    s32                       input_dst_cpuid;
    s32                       output_dst_cpuid;
    struct rpmsg_irq_data     irq_data;
    int                       base_vq_id;
    int                       num_of_vqs;
    struct kobject            kobj;
    __u32                     tx_count;
    __u32                     rx_count;
    int                       log_level;
    struct rpmsg_channel_info chinfo;
    struct delayed_work       delay_work_info;
    struct list_head          extra_endpoint_list;
    u64                       features;
    unsigned int              rpmsg_num_bufs;
    unsigned int              rpmsg_buf_size;
};

struct nokia_rpmsg_vq_info {
    __u16                     num;       // num of entries in the virtio_ring
    __u16                     vq_id;     // a globaly unique index of this virtqueue
    void*                     addr;      // address where we mapped the virtio ring
    struct nokia_rpmsg_vproc* rpdev;
};

// the alignment between the consumer and producer parts of the vring
// note: this is part of the "wire" protocol, if you change this, you need to update the bios image as well
#define RPMSG_VRING_ALIGN	 (4096)
#define DEF_RPMSG_NUM_BUFS   (256)

// with 256 buffers, our vring will occupy 3 pages
#define DEF_RPMSG_RING_SIZE	((DIV_ROUND_UP(vring_size(DEF_RPMSG_NUM_BUFS/2, RPMSG_VRING_ALIGN), PAGE_SIZE)) * PAGE_SIZE)
#define to_nokia_rpdev(vd) container_of(vd, struct nokia_rpmsg_vproc, vdev)

// management functions

void rpmsg_plat_info_release(struct kobject* kobj){}

ssize_t rpmsg_plat_info_show(struct kobject* kobj, struct attribute* attr, char* buf){
    int len = 0;
    struct nokia_rpmsg_vproc* rpdev = container_of(kobj, struct nokia_rpmsg_vproc, kobj);
    buf[0] = '\0';
    if(strcmp(attr->name, "version") == 0){
        len = sprintf(buf, "%s\n", DRIVER_VERSION);
    }
    else if(strcmp(attr->name, "counters") == 0){
        len = sprintf(buf, "tx:%u, rx:%u\n", rpdev->tx_count, rpdev->rx_count);
    }
    return len;
}

ssize_t rpmsg_plat_info_store(struct kobject* kobj, struct attribute* attr, const char* buf, size_t count){
    unsigned long value = 0;
    struct nokia_rpmsg_vproc* rpdev = container_of(kobj, struct nokia_rpmsg_vproc, kobj);
    value = simple_strtoul(buf, NULL, 10);
    if(strcmp(attr->name, "loglevel") == 0){
        printk("Set log level to 0x%lX.\n", value);
        rpdev->log_level = value;
    }
    return count;
}

static struct attribute version_attr = SYSFS_ATTR(version, S_IRUGO);
static struct attribute counters_attr = SYSFS_ATTR(counters, S_IRUGO);
static struct attribute loglevel_attr = SYSFS_ATTR(loglevel, S_IWUSR);

static struct attribute* rpmsg_plat_info_attrs[] = {
    &version_attr,
    &counters_attr,
    &loglevel_attr,
    NULL
};

static struct sysfs_ops rpmsg_plat_info_sysfs_ops = {
    .show = rpmsg_plat_info_show,
    .store = rpmsg_plat_info_store
};

static struct kobj_type rpmsg_plat_info_kobj_type = {
    .release = rpmsg_plat_info_release,
    .sysfs_ops = &rpmsg_plat_info_sysfs_ops,
    .default_attrs = rpmsg_plat_info_attrs,
};

#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 9, 0)
static u32 nokia_rpmsg_get_features(struct virtio_device* vdev){
#else
static u64 nokia_rpmsg_get_features(struct virtio_device* vdev){
#endif
    struct nokia_rpmsg_vproc* rpdev = to_nokia_rpdev(vdev);
    return rpdev->features;
    // return 1 << VIRTIO_RPMSG_F_NS;
}

#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 9, 0)
static void nokia_rpmsg_finalize_features(struct virtio_device* vdev){
    vring_transport_features(vdev);            // give virtio_ring a chance to accept features
}
#else
static int nokia_rpmsg_finalize_features(struct virtio_device* vdev){
    vring_transport_features(vdev);            // give virtio_ring a chance to accept features
    return 0;
}
#endif

// kick the remote processor, and let it know which virtqueue to poke at
#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 9, 0)
static void nokia_rpmsg_notify(struct virtqueue* vq){
#else
static bool nokia_rpmsg_notify(struct virtqueue* vq){
#endif
    struct nokia_rpmsg_vq_info* rpvq= vq->priv;
    struct virtio_device* vdev = &rpvq->rpdev->vdev;
    int ret;

    // see blog post: issue BBN-62211
    // in network tx context(__dev_queue_xmit), spinlock is used to disable preempt.
    // mutex cannot be used here, may cause cpu schedule issue.
    spin_lock(&rpvq->rpdev->spin_lock);

    // message payload could be virtqueue id, now no use as there is only one queue for tx or rx
    ret = rpmsg_irq_trigger(rpvq->rpdev->irq_data.output_irq_num, rpvq->rpdev->irq_data.channel_num);
    rpvq->rpdev->tx_count++;
    if(rpvq->rpdev->log_level > LOG_NONE){
        dev_info(&vdev->dev, "notify to remote, txcount = %u !\n", rpvq->rpdev->tx_count);
    }
    spin_unlock(&rpvq->rpdev->spin_lock);
    if(ret){
        pr_err("nokia_rpmsg_notify() failed: %d\n", ret);
    }
#if LINUX_VERSION_CODE > KERNEL_VERSION(4, 9, 0)
    return !ret;
#endif
}

static struct virtqueue* rp_find_vq(struct virtio_device* vdev, unsigned index,
                                    void (*callback) (struct virtqueue* vq), const char* name){
    struct nokia_rpmsg_vproc* rpdev = to_nokia_rpdev(vdev);
    struct nokia_rpmsg_vq_info* rpvq;
    struct virtqueue* vq;
    int err;
    unsigned int rpmsg_bufmem_size = 0, buf_nums = 0;

    if(rpdev->rpmsg_num_bufs > 0){
        buf_nums = rpdev->rpmsg_num_bufs;
        rpmsg_bufmem_size = ((DIV_ROUND_UP(vring_size(rpdev->rpmsg_num_bufs / 2,
            RPMSG_VRING_ALIGN), PAGE_SIZE)) * PAGE_SIZE);
        dev_dbg(&vdev->dev, "Use dynamic rpmsg buffer(%d) and size(%d),
            rpmsg_bufmem_size = 0x%X\n", rpdev->rpmsg_num_bufs, rpdev->rpmsg_buf_size, rpmsg_bufmem_size);
    }
    else{
        buf_nums = DEF_RPMSG_NUM_BUFS;
        rpmsg_bufmem_size = DEF_RPMSG_RING_SIZE;
        dev_dbg(&vdev->dev, "Use fixed rpmsg buffer and size,
            rpmsg_bufmem_size = 0x%X, buf_nums = 0x%X\n", rpmsg_bufmem_size, buf_nums);
    }
    rpvq = kmalloc(sizeof(*rpvq), GFP_KERNEL);
    if(!rpvq){
        return ERR_PTR(-ENOMEM);
    }

    // ioremap normal memory, so we cast away sparse's complaints */
#ifdef CONFIG_ARCH_NOKIA_GIN
    rpvq->addr = (__force void *) ioremap_wc(rpdev->vring[index], rpmsg_bufmem_size);
#else
    rpvq->addr = (__force void *) ioremap_nocache(rpdev->vring[index], rpmsg_bufmem_size);
#endif
    if(!rpvq->addr){
        err = -ENOMEM;
        goto free_rpvq;
    }
    if(rpdev->rpmsg_role == RPMSG_MASTER){
        memset_io(rpvq->addr, 0, rpmsg_bufmem_size);
    }
    //dev_info(&vdev->dev, "vring%d: phys 0x%llX, virt 0x%lX, rpmsg_bufmem_size = 0x%lX\n", index, rpdev->vring[index], (unsigned long) rpvq->addr, rpmsg_bufmem_size);

#if LINUX_VERSION_CODE < KERNEL_VERSION(5, 4, 0)
    vq = vring_new_virtqueue(index, buf_nums/2, RPMSG_VRING_ALIGN, vdev, true, rpvq->addr, nokia_rpmsg_notify, callback, name);
#else
    vq = vring_new_virtqueue(index, buf_nums/2, RPMSG_VRING_ALIGN, vdev, true, NULL, rpvq->addr, nokia_rpmsg_notify, callback, name);
#endif
    if(!vq){
        dev_err(&vdev->dev, "vring_new_virtqueue failed\n");
        err = -ENOMEM;
        goto unmap_vring;
    }
    rpdev->vq[index] = vq;
    vq->priv = rpvq;
    rpvq->vq_id = rpdev->base_vq_id + index;       // system-wide unique id for this virtio que
    rpvq->rpdev = rpdev;
    spin_lock_init(&rpdev->spin_lock);
    return vq;

unmap_vring:
    iounmap((__force void __iomem*) rpvq->addr);   // iounmap normal memory, make sparse happy
free_rpvq:
    kfree(rpvq);
    return ERR_PTR(err);
}

static void nokia_rpmsg_del_vqs(struct virtio_device* vdev){
    struct virtqueue* vq;
    struct virtqueue* n;
    list_for_each_entry_safe(vq, n, &vdev->vqs, list){
        struct nokia_rpmsg_vq_info* rpvq = vq->priv;
        iounmap(rpvq->addr);
        vring_del_virtqueue(vq);
        kfree(rpvq);
    }
}

#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 9, 0)
static int nokia_rpmsg_find_vqs(struct virtio_device* vdev, unsigned nvqs, struct virtqueue* vqs[],
                                vq_callback_t* callbacks[], const char* names[]){
#else
#if LINUX_VERSION_CODE < KERNEL_VERSION(5, 4, 0)
static int nokia_rpmsg_find_vqs(struct virtio_device* vdev, unsigned nvqs, struct virtqueue* vqs[],
                                vq_callback_t* callbacks[], const char* const names[]){
#else
static int nokia_rpmsg_find_vqs(struct virtio_device* vdev, unsigned nvqs, struct virtqueue* vqs[],
                                vq_callback_t* callbacks[], const char* const names[], const bool* ctx,
                                struct irq_affinity* desc){
#endif
#endif
    struct nokia_rpmsg_vproc* rpdev = to_nokia_rpdev(vdev);
    int i, err;
    if(nvqs != VQUEUE_NUM_PER_VIRTDEV){          // maintain 2 virtio que per remote processor (rx/tx)
        return -EINVAL;
    }
    for(i = 0; i < nvqs; ++i){
        vqs[i] = rp_find_vq(vdev, i, callbacks[i], names[i]);
        if(IS_ERR(vqs[i])){
            err = PTR_ERR(vqs[i]);
            goto error;
        }
    }
    rpdev->num_of_vqs = nvqs;
    return 0;
error:
    nokia_rpmsg_del_vqs(vdev);
    return err;
}

static void nokia_rpmsg_get(struct virtio_device* vdev, unsigned offset, void* buf, unsigned len){
    struct nokia_rpmsg_vproc* rpdev = to_nokia_rpdev(vdev);
    struct rpmsg_channel_info* pChinfo = NULL;

    if(!buf || !len || !rpdev || !vdev){
        return;
    }
    if(offset & GET_ROLE_INFO_CONFIG_OFFSET){
        if(len >= sizeof(rpdev->rpmsg_role)){
            memcpy(buf, &(rpdev->rpmsg_role), sizeof(rpdev->rpmsg_role));
        }
    }
    else if(offset & GET_CHANNEL_INFO_CONFIG_OFFSET){
        if(len >= sizeof(struct rpmsg_channel_info)){
            pChinfo = (struct rpmsg_channel_info*) buf;
            pChinfo->dst = rpdev->chinfo.dst;
            pChinfo->src = rpdev->chinfo.src;
            strcpy(pChinfo->name, rpdev->chinfo.name);
            pChinfo->p_extra_endpoint_list = rpdev->chinfo.p_extra_endpoint_list;
        }
    }
    else if(offset == GET_DATA_POOL_ADDR_OFFSET){
        if(len >= sizeof(u64)){
            memcpy(buf, &(rpdev->data_buffer_pool), sizeof(u64));
        }
    }
    else if (offset == GET_RPMSG_BUFFER_SIZE_INFO) {
        if(len >= sizeof(rpdev->rpmsg_buf_size)){
            memcpy(buf, &(rpdev->rpmsg_buf_size), sizeof(rpdev->rpmsg_buf_size));
        }
    }
    else if(offset == GET_RPMSG_BUFFER_NUM_INFO){
        if(len >= sizeof(rpdev->rpmsg_num_bufs)){
            memcpy(buf, &(rpdev->rpmsg_num_bufs), sizeof(rpdev->rpmsg_num_bufs));
        }
    }
}

static void nokia_rpmsg_reset(struct virtio_device* vdev){
    dev_dbg(&vdev->dev, "reset !\n");
}

static u8 nokia_rpmsg_get_status(struct virtio_device* vdev){
    return 0;
}

static void nokia_rpmsg_set_status(struct virtio_device* vdev, u8 status){
    dev_dbg(&vdev->dev, "%s new status: %d\n", __func__, status);
}

static void nokia_rpmsg_vproc_release(struct device* dev){
    /* this handler is provided so driver core doesn't yell at us */
}

struct virtio_config_ops nokia_rpmsg_config_ops = {
    .get_features = nokia_rpmsg_get_features,
    .finalize_features = nokia_rpmsg_finalize_features,
    .find_vqs = nokia_rpmsg_find_vqs,
    .del_vqs = nokia_rpmsg_del_vqs,
    .reset = nokia_rpmsg_reset,
    .set_status = nokia_rpmsg_set_status,
    .get_status = nokia_rpmsg_get_status,
    .get = nokia_rpmsg_get,
};

static void rpmsg_vproc_initialize(struct nokia_rpmsg_vproc* rpdev){
    if(!rpdev){
        return;
    }
    memset(rpdev, 0, sizeof(struct nokia_rpmsg_vproc));
    rpdev->vdev.id.device = VIRTIO_ID_RPMSG;
    rpdev->vdev.config = &nokia_rpmsg_config_ops;
    rpdev->vdev.dev.release = nokia_rpmsg_vproc_release;
    rpdev->features = (1 << VIRTIO_RING_F_FIXED_DATA_BUFFER);
    INIT_LIST_HEAD(&rpdev->extra_endpoint_list);
}

static const struct of_device_id nokia_rpmsg_dt_ids[] = {
    {.compatible = "fsl,nokia-rpmsg",},
    {.compatible = "nokia-rpmsg",},
    { /*any others */ }
};

MODULE_DEVICE_TABLE(of, nokia_rpmsg_dt_ids);

static void rpmsg_work_handler(struct work_struct* pWork){
    struct delayed_work* pDwork = container_of(pWork, struct delayed_work, work);
    struct nokia_rpmsg_vproc* rpdev = container_of(pDwork, struct nokia_rpmsg_vproc, delay_work_info);
    struct virtio_device* vdev = &rpdev->vdev;

    rpdev->rx_count++;
    if(rpdev->log_level > LOG_NONE){
        dev_info(&vdev->dev, "get one rpmsg, rxcount = %u !\n", rpdev->rx_count);
    }
    // message payload could be virtqueue id, now no use as there is only one queue for tx or rx, set as 0
    vring_interrupt(RPMSG_REMOTE == rpdev->rpmsg_role, rpdev->vq[VIRTQUEUE_RX]);
}

static int nokia_rpmsg_probe_dt(struct platform_device* pdev, struct nokia_rpmsg_vproc* rpdev, int index){
    const char* channame;
    int channame_len = 0;
    int eth_endpoints, misc_endpoints;
    struct endpoint_node_info* p_ept_node = NULL;
    struct device* dev = &pdev->dev;
    int ret = 0, i;

    struct of_get_u32_from_dts {
        char* name;
        u32*  value;
        bool  must;
    } u32_properties [] = {
        { "rpmsg-role",         &rpdev->rpmsg_role,              true },
        { "input-msgirq-num",   &rpdev->irq_data.input_irq_num,  false },
        { "output-msgirq-num",  &rpdev->irq_data.output_irq_num, true },
        { "input-dstcpu-id",    &rpdev->input_dst_cpuid,         false },
        { "output-dstcpu-id",   &rpdev->output_dst_cpuid,        false },
        { "channel-num",        &rpdev->irq_data.channel_num,    true },
        { "channel-src",        &rpdev->chinfo.src,              true },
        { "channel-dst",        &rpdev->chinfo.dst,              true },
        { "eth-dev-endpoints",  &eth_endpoints,                  true },
        { "misc-dev-endpoints", &misc_endpoints,                 true },
        { "rpmsg-num-bufs",     &rpdev->rpmsg_num_bufs,          false },
        { "rpmsg-buf-size",     &rpdev->rpmsg_buf_size,          false },
    };

    struct of_get_u64_from_dts {
        char* name;
        u64*  value;
        bool  must;
    } u64_properties[] = {
        { "shmem-rx-vring",     &rpdev->vring[VIRTQUEUE_RX], true },
        { "shmem-tx-vring",     &rpdev->vring[VIRTQUEUE_TX], true },
        { "data-buffer-pool",   &rpdev->data_buffer_pool, false },
    };

    rpdev->input_dst_cpuid = -1;
    rpdev->output_dst_cpuid = -1;

    for(i = 0; i < ARRAY_SIZE(u32_properties); i++){
        ret = of_property_read_u32_index(dev->of_node, u32_properties[i].name, index, u32_properties[i].value);
        if(ret && u32_properties[i].must){
            dev_err(dev, "can't get %s\n", u32_properties[i].name);
            return -ENXIO;
        }
        dev_dbg(dev, "%s is 0x%x\n", u32_properties[i].name, *u32_properties[i].value);
    }

    for(i = 0; i < ARRAY_SIZE(u64_properties); i++){
        ret = of_property_read_u64_index(dev->of_node, u64_properties[i].name, index, u64_properties[i].value);
        if(ret && u64_properties[i].must){
            dev_err(dev, "can't get %s\n", u64_properties[i].name);
            return -ENXIO;
        }
        dev_dbg(dev, "%s is 0x%llx\n", u64_properties[i].name, *u64_properties[i].value);
    }

    // 1 for standard rpmsg, only the remote side (rtos) would create the first channel.
    // remote side would use name service to annonce its channel and then master side
    // would get the channel name to creat the same channel and endpoints.
    // 2 for isam, we creat same default channels on both sides directly.
    // name service message could still be handled for other channels.

    ret = of_property_read_string_index(dev->of_node, "channel-name", index, &channame);
    channame_len = strlen(channame);
    if(channame_len >= RPMSG_NAME_SIZE - 1){
        dev_err(dev, "channel name is too long (%d >= %d)\n", channame_len, RPMSG_NAME_SIZE - 1);
        return -ENXIO;
    }
    memcpy(rpdev->chinfo.name, channame, channame_len);
    rpdev->chinfo.name[channame_len] = '\0';
    dev_dbg(dev, "Got rpmsg channel-name: %s\n", rpdev->chinfo.name);

    if(rpdev->data_buffer_pool){
#if defined(VIRTIO_RING_F_DATA_BUFFER_NOT_DIRECTMAP)
#if defined(CONFIG_LOWMEM_SIZE)
        if(rpdev->data_buffer_pool > CONFIG_LOWMEM_SIZE){
            rpdev->features |= (1 << VIRTIO_RING_F_DATA_BUFFER_NOT_DIRECTMAP);
        }
#else
        rpdev->features |= (1 << VIRTIO_RING_F_DATA_BUFFER_NOT_DIRECTMAP);
#endif
#endif
    }
    else{
        dev_info(dev, "No data-buffer-pool offer, will use dma alloc\n");
        rpdev->data_buffer_pool = 0;
    }
    dev_dbg(dev, "Got RX vring addr: 0x%llX, TX vring addr : 0x%llX, data poll addr : 0x%llX\n",
            rpdev->vring[VIRTQUEUE_RX], rpdev->vring[VIRTQUEUE_TX], rpdev->data_buffer_pool);

    p_ept_node = devm_kzalloc(dev, sizeof(*p_ept_node), GFP_KERNEL);
    if(!p_ept_node){
        dev_err(dev, "cannot allocate memory\n");
        ret = -ENOMEM;
        goto fail_out;
    }
    p_ept_node->ept_num = eth_endpoints;
    p_ept_node->ept_ifc_type = RPMSG_EPT_ITF_ETH;
    p_ept_node->channel_num = rpdev->irq_data.channel_num;
    list_add(&p_ept_node->endpoint_list_node, &rpdev->extra_endpoint_list);

    p_ept_node = devm_kzalloc(dev, sizeof(*p_ept_node), GFP_KERNEL);
    if(!p_ept_node){
        dev_err(dev, "cannot allocate memory\n");
        ret = -ENOMEM;
        goto fail_out;
    }
    p_ept_node->ept_num = misc_endpoints;
    p_ept_node->ept_ifc_type = RPMSG_EPT_ITF_MISC;
    p_ept_node->channel_num = rpdev->irq_data.channel_num;
    list_add(&p_ept_node->endpoint_list_node, &rpdev->extra_endpoint_list);

    rpdev->chinfo.p_extra_endpoint_list = &rpdev->extra_endpoint_list;
    return 0;
fail_out:
    list_for_each_entry(p_ept_node, &rpdev->extra_endpoint_list, endpoint_list_node){
        devm_kfree(dev, p_ept_node);
    }
    return ret;
}

static int nokia_rpmsg_probe(struct platform_device* pdev){
    int ret = 0;
    int input_dst_cpuid = -1;
    struct nokia_rpmsg_vproc* rpdev = NULL;
    struct device* dev = &pdev->dev;
    struct rpmsg_irq_data output_irqdata = {0};
    int i, channel_count;

    channel_count = of_property_count_u32_elems(dev->of_node, "channel-num");
    if(channel_count < 0){
        dev_err(dev, "No 'channel-num' property found\n");
        return -EINVAL;
    }

    rpdev = devm_kcalloc(dev, channel_count, sizeof(*rpdev), GFP_KERNEL);
    if(!rpdev){
        dev_err(dev, "cannot allocate msg_irq resource\n");
        return -ENOMEM;
    }

    for(i = 0; i < channel_count; i++, rpdev++){
        rpmsg_vproc_initialize(rpdev);
        ret = nokia_rpmsg_probe_dt(pdev, rpdev, i);
        if(ret < 0){
            goto einval_out;
        }
        rpdev->vdev.dev.parent = dev;
        ret = register_virtio_device(&rpdev->vdev);
        if(ret){
            dev_err(dev, "%s failed to register rpdev: %d\n", __func__, ret);
            goto einval_out;
        }
        ret = kobject_init_and_add(&rpdev->kobj, &rpmsg_plat_info_kobj_type,
                                   &dev->kobj, "rpmsg_plat_info%d",
                                   rpdev->irq_data.channel_num);
        if(ret){
            dev_err(dev, "cannot add kobject resource\n");
            goto einval_out;
        }

        INIT_DELAYED_WORK(&(rpdev->delay_work_info), rpmsg_work_handler);
        if(rpdev->input_dst_cpuid >= 0){
            input_dst_cpuid = rpdev->input_dst_cpuid;
        }
        else{
            if(rpdev->rpmsg_role == RPMSG_MASTER){
                input_dst_cpuid = DST_CPU_1;
            }
            else{
                input_dst_cpuid = DST_CPU_0;
            }
        }

        rpdev->irq_data.irq_index = rpdev->irq_data.input_irq_num;
        rpdev->irq_data.descpu = input_dst_cpuid;
        rpdev->irq_data.pDwork = &(rpdev->delay_work_info);
        rpdev->irq_data.irq = platform_get_irq(pdev, i);
        rpdev->irq_data.channel_name = rpdev->chinfo.name;
        rpdev->irq_data.dev = dev;
        rpmsg_irq_setup(&rpdev->irq_data);

        if(rpdev->output_dst_cpuid >= 0){
            output_irqdata.irq_index = rpdev->irq_data.output_irq_num;
            output_irqdata.channel_num = rpdev->irq_data.channel_num;
            output_irqdata.descpu = rpdev->output_dst_cpuid;
            rpmsg_irq_setup(&output_irqdata);
        }
    }
    return ret;
einval_out:
    devm_kfree(dev, rpdev);
    return -EINVAL;
}

static int nokia_rpmsg_remove(struct platform_device* pdev){
    struct device* dev = &pdev->dev;
    struct virtio_device* vdev = dev_to_virtio(dev);
    struct nokia_rpmsg_vproc* rpdev = to_nokia_rpdev(vdev);

    cancel_delayed_work_sync(&(rpdev->delay_work_info));
    unregister_virtio_device(vdev);
    devm_kfree(dev, rpdev);
    return 0;
}

static struct platform_driver nokia_rpmsg_driver = {
    .driver = {
        .owner = THIS_MODULE,
        .name = "nokia-rpmsg",
        .of_match_table = nokia_rpmsg_dt_ids,
    },
    .probe = nokia_rpmsg_probe,
    .remove = nokia_rpmsg_remove,
};

static int __init nokia_rpmsg_init(void){
    int ret;
    ret = platform_driver_register(&nokia_rpmsg_driver);
    if(ret){
        pr_err("Unable to initialize rpmsg driver\n");
    }
    return ret;
}

static void __exit nokia_rpmsg_exit(void){
    pr_info("nokia rpmsg driver unregistered.\n");
    platform_driver_unregister(&nokia_rpmsg_driver);
}

module_exit(nokia_rpmsg_exit);
module_init(nokia_rpmsg_init);

MODULE_AUTHOR("Wu Stan <stan.wu@nokia-sbell.com.cn>");
MODULE_DESCRIPTION("Nokia rpmsg platform driver");
MODULE_LICENSE("GPL");
MODULE_VERSION(DRIVER_VERSION);
```

2 the rpmsg driver code for different interfaces, at isam-linux-drivers/misc/rpmsg_interface.c:

```blurtext
#define DRIVER_VERSION      "0.0.1"

// 0.0.1   Wu Stan             Dec 25 2018
// Init version of rpmsg driver for different interfaces

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/virtio.h>
#include <linux/rpmsg.h>
#include <linux/slab.h>
#include <linux/device.h>
#include <linux/mutex.h>
#include <linux/fs.h>
#include <linux/errno.h>
#include "rpmsg_interface.h"

#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 9, 0)
static void rpmsg_ifc_drv_cb(struct rpmsg_channel *rpdev, void *data, int len, void *priv, u32 src)
#else
static int rpmsg_ifc_drv_cb(struct rpmsg_device *rpdev, void *data, int len, void *priv, u32 src)
#endif
{
	//Callback for default endpoint, should only come here where there are no other endpoints
	dev_err(&rpdev->dev, "ERROR: %s %d\n", __FUNCTION__, __LINE__);
#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 9, 0)
#else
    return 0;
#endif
}

#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 9, 0)
static int rpmsg_ifc_drv_probe(struct rpmsg_channel *rpdev)
#else
static int rpmsg_ifc_drv_probe(struct rpmsg_device *rpdev)
#endif
{
	struct list_head *p_extra_endpoint_list = rpdev->ept->priv;
	struct endpoint_node_info *p_ept_node = NULL;
	int ret = 0;

	dev_dbg(&rpdev->dev, "new channel: 0x%x -> 0x%x!\n", rpdev->src, rpdev->dst);
	if (p_extra_endpoint_list) {
		list_for_each_entry(p_ept_node, p_extra_endpoint_list, endpoint_list_node) {
			dev_dbg(&rpdev->dev, "extra endpoint: num: 0x%x, type: 0x%x!\n", p_ept_node->ept_num, p_ept_node->ept_ifc_type);
#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 9, 0)
			p_ept_node->rpmsg_chnl = rpdev;
#else
			p_ept_node->rpdev = rpdev;
#endif
			if (p_ept_node->ept_ifc_type == RPMSG_EPT_ITF_MISC) {
				ret = rpmsg_register_miscdev(p_ept_node);
			} else if (p_ept_node->ept_ifc_type == RPMSG_EPT_ITF_ETH) {
				ret = rpmsg_register_ethdev(p_ept_node);
			}

			if (ret)
				dev_err(&rpdev->dev, "Failed with extra endpoint: num: 0x%x, type: 0x%x!\n", p_ept_node->ept_num, p_ept_node->ept_ifc_type);
		}
	} else
		dev_info(&rpdev->dev, "No extra endpoint for this channel!\n");
	return 0;
}

#if LINUX_VERSION_CODE < KERNEL_VERSION(4, 9, 0)
static void rpmsg_ifc_drv_remove(struct rpmsg_channel *rpdev)
#else
static void rpmsg_ifc_drv_remove(struct rpmsg_device *rpdev)
#endif
{
	struct list_head *p_extra_endpoint_list = rpdev->ept->priv;
	struct endpoint_node_info *p_ept_node = NULL;

	if (p_extra_endpoint_list) {
		list_for_each_entry(p_ept_node, p_extra_endpoint_list, endpoint_list_node) {
			dev_dbg(&rpdev->dev, "extra endpoint: num: 0x%x, type: 0x%x!\n", p_ept_node->ept_num, p_ept_node->ept_ifc_type);
			if (p_ept_node->ept_ifc_type == RPMSG_EPT_ITF_MISC) {
				rpmsg_remove_miscdev(p_ept_node);
			} else if (p_ept_node->ept_ifc_type == RPMSG_EPT_ITF_ETH) {
				rpmsg_remove_ethdev(p_ept_node);
			}
			if (p_ept_node->p_ept)
				rpmsg_destroy_ept(p_ept_node->p_ept);
		}
	} else
		dev_info(&rpdev->dev, "No extra endpoint for this channel!\n");
}

static struct rpmsg_device_id rpmsg_ifc_drv_id_table[] = {
	{.name = "nokia-fsproxy-channel"},
	{.name = "nokia-eqpthwa-channel"},
	{.name = "nokia-redundancy-channel"},
	{.name = "nokia-swm1-channel"},
	{.name = "nokia-swm2-channel"},
	{.name = "nokia-nat-channel"},
	{.name = "nokia-debug-channel"},
	{.name = "nokia-clk1-channel"},
	{.name = "nokia-clk2-channel"},
	{.name = "nokia-typeb1-channel"},
	{.name = "nokia-typeb2-channel"},
	{.name = "nokia-typeb3-channel"},
	{.name = "nokia-swm-ntio-channel"},
	{.name = "nokia-x2-channel"},
	{.name = "nokia-x3-channel"},
	{.name = "nokia-x4-channel"},
	{.name = "nokia-x5-channel"},
	{},
};

static struct rpmsg_driver rpmsg_interface_drv = {
	.drv.name = "rpmsg_ifc_drv",
	.drv.owner = THIS_MODULE,
	.id_table = rpmsg_ifc_drv_id_table,
	.probe = rpmsg_ifc_drv_probe,
	.remove = rpmsg_ifc_drv_remove,
	.callback = rpmsg_ifc_drv_cb,
};

static int __init rpmsg_ifcdrv_init(void){
	int err = 0;
	if((err = register_rpmsg_driver(&rpmsg_interface_drv)) != 0){
		pr_err("ERROR: %s %d rc=%d\n", __FUNCTION__, __LINE__, err);
	}
	return err;
}

static void __exit rpmsg_ifcdrv_exit(void){
    unregister_rpmsg_driver(&rpmsg_interface_drv);
}

module_init(rpmsg_ifcdrv_init);
module_exit(rpmsg_ifcdrv_exit);

MODULE_AUTHOR("Wu Stan <stan.wu@nokia-sbell.com.cn>");
MODULE_DESCRIPTION("Nokia rpmsg driver for different interfaces");
MODULE_LICENSE("GPL");
MODULE_VERSION(DRIVER_VERSION);
```

<hr>

### # the device description info (buildroot)

```blurtext
./board/Nokia/isam-reborn/board/lant-a/lanta.dts
./board/Nokia/isam-reborn/board/fant-f-y/fant-f-y-3041.dts
./board/Nokia/isam-reborn/board/fant-f-y/fant-f-y-4040.dts
./board/Nokia/isam-reborn/board/fant-g-y/fanth-y.dts
./board/Nokia/isam-reborn/board/fant-g-y/fantg-y.dts
```

rpmsg device dts in buildroot/board/Nokia/isam-reborn/board/lant-a/lanta.dts:

```blurtext
@soc {
    ...
	rpmsg{
		#address-cells = <0x2>;
		#size-cells = <0>;

		compatible = "fsl,nokia-rpmsg";
		input-msgirq-num = <3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3>;  /* all channel use msg irq 3 */
		output-msgirq-num = <7 7 7 7 7 7 7 7 7 7 7 7 7 7 7 7 7>; /* all channel use msg irq 7 */
		input-dstcpu-id = <0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0>;
		output-dstcpu-id = <2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2>;
		rpmsg-role = <1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1>;        /* 1:MASTER, 2:REMOTE */
		channel-num = <0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16>;
		channel-src = <0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16>;
		channel-dst = <0x100 0x200 0x300 0x400 0x500 0x600 0x700 0x800 0x900 0xA00 0xB00 0xC00 0xD00 0xE00 0xF00 0x1000 0x1100 0x1200>;
		channel-name =  "nokia-fsproxy-channel", "nokia-eqpthwa-channel",
                        "nokia-redundancy-channel", "nokia-swm1-channel",
                        "nokia-swm2-channel", "nokia-nat-channel",
                        "nokia-debug-channel", "nokia-clk1-channel",
                        "nokia-clk2-channel", "nokia-typeb1-channel",
                        "nokia-typeb2-channel", "nokia-typeb3-channel",
                        "nokia-swm-ntio-channel", "nokia-x2-channel",
                        "nokia-x3-channel", "nokia-x4-channel",
                        "nokia-x5-channel";
		eth-dev-endpoints = <0x10 0x20 0x30 0x40 0x50 0x60 0x70 0x80 0x90 0xA0 0xB0 0xC0 0xD0 0xE0 0xF0 0x100 0x110>;
		misc-dev-endpoints = <0x12 0x22 0x32 0x42 0x52 0x62 0x72 0x82 0x92 0xA2 0xB2 0xC2 0xD2 0xE2 0xF2 0x102 0x112>;
		shmem-rx-vring   = <0x0 0xB8000000 0x0 0xB8210000 0x0 0xB8260000 0x0 0xB82B0000
                            0x0 0xB8300000 0x0 0xB8350000 0x0 0xB83A0000 0x0 0xB83F0000
                            0x0 0xB8440000 0x0 0xB8490000 0x0 0xB84E0000 0x0 0xB8530000
                            0x0 0xB8580000 0x0 0xB85D0000 0x0 0xB8620000 0x0 0xB8670000
                            0x0 0xB86C0000>;
		shmem-tx-vring   = <0x0 0xB8008000 0x0 0xB8218000 0x0 0xB8268000 0x0 0xB82B8000
                            0x0 0xB8308000 0x0 0xB8358000 0x0 0xB83A8000 0x0 0xB83F8000
                            0x0 0xB8448000 0x0 0xB8498000 0x0 0xB84E8000 0x0 0xB8538000
                            0x0 0xB8588000 0x0 0xB85D8000 0x0 0xB8628000 0x0 0xB8678000
                            0x0 0xB86C8000>;
		data-buffer-pool = <0x0 0xB8010000 0x0 0xB8220000 0x0 0xB8270000 0x0 0xB82C0000
                            0x0 0xB8310000 0x0 0xB8360000 0x0 0xB83B0000 0x0 0xB8400000
                            0x0 0xB8450000 0x0 0xB84A0000 0x0 0xB84F0000 0x0 0xB8540000
                            0x0 0xB8590000 0x0 0xB85E0000 0x0 0xB8630000 0x0 0xB8680000
                            0x0 0xB86D0000>;
		rpmsg-num-bufs   = <256 512 512 512 512 512 512 512 512 512 512 512 512 512 512 512 512>;
		rpmsg-buf-size   = <8192 512 512 512 512 512 512 512 512 512 512 512 512 512 512 512 512>;
		status = "okay";
	};

    ...

    msgirq3 {
        compatible = "alu,msgirq";
        msg-num = <0x3>;
        mivpr-offset = <0x51660>;
        midr-offset = <0x51670>;
        rpmsg-use = <0x1>;
    };

    ...

    msgirq7 {
        compatible = "alu,msgirq";
        msg-num = <0x7>;
        mivpr-offset = <0x516e0>;
        midr-offset = <0x516f0>;
        rpmsg-use = <0x1>;
    };
};
```

<hr>

### # the platform basic infra utilities for rpmsg (sw)

```blurtext
sw:
./vobs/dsl/sw/y/src/yAPI/src/yipc.c
./vobs/dsl/sw/y/src/yAPI/src/yipc_rpmsg.c
./vobs/dsl/sw/y/src/yAPI/src/yipc_send_recv.c
./vobs/dsl/sw/y/src/yAPI/src/yipc_uds.c
./vobs/dsl/sw/y/src/yAPI/src/yipc_ysocket.c
```
