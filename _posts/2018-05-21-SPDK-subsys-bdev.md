---
layout: post
title: 【SPDK】五、bdev子系统
date: 2018-05-21 
tags: SPDK
---

&emsp;&emsp;SPDK从功能角度将各个独立的部分划分为“**子系统**“。例如对各种后端存储的访问属于bdev子系统，又例如对虚拟机呈现各种设备属于vhost子系统。不同场景下，各种工具可以通过组合不同的子系统来实现各种不同的功能。例如虚拟化场景下，vhost主要集成了bdev、vhost、scsi等子系统。这些子系统存在一定依赖关系，例如vhost子系统依赖bdev，这就需要将被依赖的子系统先初始化完成，才能执行其它子系统的初始化。

&emsp;&emsp;本篇博文我们先整体介绍一下SPDK子系统的初始化流程，然后再深入分析一下bdev子系统。vhost子系统我们将在独立的博文中展开分析。

### SPDK子系统

&emsp;&emsp;通过前文的分析，我们知道主线程在执行_spdk_reactor_run时，首先处理的事件便是verify事件，该事件处理函数为spdk_subsystem_verify：

```
spdk/lib/event/subsystem.c:

static void
spdk_subsystem_verify(void *arg1, void *arg2)
{
    struct spdk_subsystem_depend *dep;

    /* 检查当前应用中所有需要的子系统及其依赖系统是否均已成功注册 */
    /* Verify that all dependency name and depends_on subsystems are registered */
    TAILQ_FOREACH(dep, &g_subsystems_deps, tailq) {
        if (!spdk_subsystem_find(&g_subsystems, dep->name)) {
            SPDK_ERRLOG("subsystem %s is missing\n", dep->name);
            spdk_app_stop(-1);
            return;
        }
        if (!spdk_subsystem_find(&g_subsystems, dep->depends_on)) {
            SPDK_ERRLOG("subsystem %s dependency %s is missing\n",
                dep->name, dep->depends_on);
            spdk_app_stop(-1);
            return;
        }
    }

    /* 按依赖关系对所有子系统进行排序 */
    subsystem_sort();

    /* 依据排序依次执行各个子系统的init函数 */
    spdk_subsystem_init_next(0);
}
```

### bdev子系统

&emsp;&emsp;bdev和vhost是虚拟化场景下两个最为主要的子系统，且vhost依赖bdev，因此我们先来分析一下bdev子系统。

&emsp;&emsp;我们可以看到bdev子系统的初始化函数为spdk_bdev_subsystem_initialize：

```
spdk/lib/event/subsystems/bdev/bdev.c:

static struct spdk_subsystem g_spdk_subsystem_bdev = {
    .name = "bdev",
    .init = spdk_bdev_subsystem_initialize,
    .fini = spdk_bdev_subsystem_finish,
    .config = spdk_bdev_config_text,
    .write_config_json = _spdk_bdev_subsystem_config_json,
};
```

&emsp;&emsp;bdev子系统针对不同的后端存储设备实现了不同的“**模块**”，例如nvme模块主要实现了用户态对nvme设备的访问操作，virtio实现了用户态对virtio设备的访问操作，又例如malloc模块通过内存实现了一个模拟的块设备。因此bdev子系统在初始化时主要针对配置文件中已经配置的后端存储模块进行初始化操作。

&emsp;&emsp;另外，bdev借助IO Channel的概念也实现了系统级的management_channel和模块级的module_channel。我们知道IO Channel是一个线程相关的概念，management_channel和module_channel也是如此：

* management_channel是线程唯一的一个对象，不同线程具备不同的的management_channel，同一个线程只有一个。目前management_channel中实现了一个线程内部独立的内存池，用来缓存bdev_io对象；

* module_channel是线程内部属于同一个模块的bdev所共享的一个对象，用来记录同一线程中属于同一模块的所有对象。例如同一个线程如果操作两个nvme的bdev对象且这两个bdev属于不同的nvme控制器，那么虽然这两个bdev对应不同的NVMe IO Channel，但是它们属于同一个module_channel。目前module_channel只含有一个模块级的引用计数和内存不足时的bdev io临时队列(当有内存空间时，实现IO重发)。

&emsp;&emsp;每个模块都会提供一个module_init函数，当bdev子系统初始化时会依次调用这些初始化函数。下面我们以NVMe和virtio两个模块为例，来简要看下模块的初始化逻辑。

#### **1. nvme模块初始化**

&emsp;&emsp;nvme模块描述如下：

```
spdk/lib/bdev/nvme/bdev_nvme.c:

static struct spdk_bdev_module nvme_if = {
    .name = "nvme",
    .module_init = bdev_nvme_library_init,
    .module_fini = bdev_nvme_library_fini,
    .config_text = bdev_nvme_get_spdk_running_config,
    .config_json = bdev_nvme_config_json,
    .get_ctx_size = bdev_nvme_get_ctx_size,

    };
```

&emsp;&emsp;这里我们可以看到nvme模块的初始化函数为bdev_nvme_library_init，另外bdev_nvme_get_ctx_size返回的context大小为nvme_bdev_io的大小。bdev子系统会以所有模块最大的context大小来创建bdev_io内存池，以此确保为所有模块申请bdev_io时都能获得足够的扩展内存(nvme_bdev_io即是对bdev_io的扩展)。

&emsp;&emsp;bdev_nvme_library_init函数从SPDK的配置文件中读取“Nvme”字段开始的相关信息，并通过这些信息创建一个NVMe控制器并获取其下的namespace，最后将namespace表示成一个bdev对象。这里我们打开看一下识别到对应NVMe控制器后的回调处理逻辑：

```

static void
attach_cb(void *cb_ctx, const struct spdk_nvme_transport_id *trid,
            struct spdk_nvme_ctrlr *ctrlr, const struct spdk_nvme_ctrlr_opts *opts)
{
    struct nvme_ctrlr *nvme_ctrlr;
    struct nvme_probe_ctx *ctx = cb_ctx;
    char *name = NULL;
    size_t i;

    /* 首先根据DPDK中PCI驱动框架识别到的NVMe控制器信息来创建一个nvme_ctrlr对象 */
    if (ctx) {
        for (i = 0; i < ctx->count; i++) {
            if (spdk_nvme_transport_id_compare(trid, &ctx->trids[i]) == 0) {
                name = strdup(ctx->names[i]);
                break;
            }
        }
    } else {
        name = spdk_sprintf_alloc("HotInNvme%d", g_hot_insert_nvme_controller_index++);
    }

    nvme_ctrlr = calloc(1, sizeof(*nvme_ctrlr));
    ...
    nvme_ctrlr->adminq_timer_poller = NULL;
    nvme_ctrlr->ctrlr = ctrlr;
    nvme_ctrlr->ref = 0;
    nvme_ctrlr->trid = *trid;
    nvme_ctrlr->name = name;

    /* 将该nvme控制器对象添加为一个io device；每个io device可申请独立的IO Channel；
        bdev_nvme_create_cb负责在IO Channel对象创建时初始化底层驱动相关对象，这里
        即是获取一个新的queue pair */
    spdk_io_device_register(ctrlr, bdev_nvme_create_cb, bdev_nvme_destroy_cb,
                                                sizeof(struct nvme_io_channel));

    /* 此处开始枚举nvme控制器下的所有namespace，并将其建为bdev对象。注意一点，此时并不会为
        bdev申请IO channel，它是vhost子系统初始时，完成线程绑定后才创建的 */
    if (nvme_ctrlr_create_bdevs(nvme_ctrlr) != 0) {
        ...
    }

    nvme_ctrlr->adminq_timer_poller = spdk_poller_register(bdev_nvme_poll_adminq, ctrlr,
                                                    g_nvme_adminq_poll_timeout_us);

    TAILQ_INSERT_TAIL(&g_nvme_ctrlrs, nvme_ctrlr, tailq);

    ...
}

/* 注意：bdev初始化时并不调用该函数 */
static int
bdev_nvme_create_cb(void *io_device, void *ctx_buf)
{
    struct spdk_nvme_ctrlr *ctrlr = io_device;
    struct nvme_io_channel *ch = ctx_buf;

    /* 分配一个nvme queue pair作为该IO Channel的实际对象 */
    ch->qpair = spdk_nvme_ctrlr_alloc_io_qpair(ctrlr, NULL, 0);
    ...
    /* 向reactor注册一个poller，轮循新分配queue pair中已完成的响应信息 */
    ch->poller = spdk_poller_register(bdev_nvme_poll, ch, 0);
    return 0;
}
```

&emsp;&emsp;类似地，我们再看一下virtio模块的初始化。

#### **2. virtio模块初始化**

&emsp;&emsp;virtio虽说起源于qemu-kvm虚拟化，但是它也是一种可用物理硬件实现的协议规范。因此SPDK也把它当做一种后端存储类型加以实现。当然，如果SPDK的vhost进程是运行在虚拟机中(而虚拟机virtio设备作为后端存储)，virtio模块就是一个必不可少的驱动模块了。

&emsp;&emsp;我们以virtio-blk设备为例，来看一下其初始化过程：

```
spdk/lib/bdev/virtio/bdev_virtio_blk.c:

static struct spdk_bdev_module virtio_blk_if = {
    .name = "virtio_blk",
    .module_init = bdev_virtio_initialize,
    .get_ctx_size = bdev_virtio_blk_get_ctx_size,
};
```
&emsp;&emsp;bdev_virtio_initialize通过配置文件获取相关配置信息，并同样借助DPDK的用户态PCI设备管理框架识别到该设备后，调用virtio_pci_blk_dev_create来创建一个virtio_blk对象：

```
spdk/lib/bdev/virtio/bdev_virtio_blk.c:

static struct virtio_blk_dev *
virtio_pci_blk_dev_create(const char *name, struct virtio_pci_ctx *pci_ctx)
{
    static int pci_dev_counter = 0;
    struct virtio_blk_dev *bvdev;
    struct virtio_dev *vdev;
    char *default_name = NULL;
    uint16_t num_queues;
    int rc;

    /* 分配一个virtio_blk_dev对象 */
    bvdev = calloc(1, sizeof(*bvdev));
    ...
    vdev = &bvdev->vdev;

    /* 为该virtio对象绑定用户态操作接口，注，该操作接口实现了virtio 1.0规范 */
    rc = virtio_pci_dev_init(vdev, name, pci_ctx);
    ...

    /* 重置设备状态 */
    rc = virtio_dev_reset(vdev, VIRTIO_BLK_DEV_SUPPORTED_FEATURES);
    ...

    /* 获取设备支持的最大队列数。如果支持多队列，从设备的配置寄存器中聊取；否则为1 */
    /* TODO: add a way to limit usable virtqueues */
    if (virtio_dev_has_feature(vdev, VIRTIO_BLK_F_MQ)) {
        virtio_dev_read_dev_config(vdev, offsetof(struct virtio_blk_config, num_queues),
            &num_queues, sizeof(num_queues));
    } else {
        num_queues = 1;
    }

    /* 初始化队列并创建bdev对象 */
    rc = virtio_blk_dev_init(bvdev, num_queues);
    ...

    return bvdev;
}

static int
virtio_blk_dev_init(struct virtio_blk_dev *bvdev, uint16_t max_queues)
{
    struct virtio_dev *vdev = &bvdev->vdev;
    struct spdk_bdev *bdev = &bvdev->bdev;
    uint64_t capacity, num_blocks;
    uint32_t block_size;
    uint16_t host_max_queues;
    int rc;

    /* 获取当前设备的块大小，默认为512字节 */
    if (virtio_dev_has_feature(vdev, VIRTIO_BLK_F_BLK_SIZE)) {
        virtio_dev_read_dev_config(vdev, offsetof(struct virtio_blk_config, blk_size),
            &block_size, sizeof(block_size));
    } else {
        block_size = 512;
    }

    /* 获取设备容量 */
    virtio_dev_read_dev_config(vdev, offsetof(struct virtio_blk_config, capacity),
        &capacity, sizeof(capacity));

    /* `capacity` is a number of 512-byte sectors. */
    num_blocks = capacity * 512 / block_size;

    /* 获取最大队列数 */
    if (virtio_dev_has_feature(vdev, VIRTIO_BLK_F_MQ)) {
            virtio_dev_read_dev_config(vdev, offsetof(struct virtio_blk_config, num_queues),
        &host_max_queues, sizeof(host_max_queues));
    } else {
        host_max_queues = 1;
    }

    if (virtio_dev_has_feature(vdev, VIRTIO_BLK_F_RO)) {
        bvdev->readonly = true;
    }

    /* bdev is tied with the virtio device; we can reuse the name */
    bdev->name = vdev->name;

    /* 按max_queues分配队列，并启动设备 */
    rc = virtio_dev_start(vdev, max_queues, 0);
    ...

    /* 为bdev对象赋值 */
    bdev->product_name = "VirtioBlk Disk";
    bdev->write_cache = 0;
    bdev->blocklen = block_size;
    bdev->blockcnt = num_blocks;

    bdev->ctxt = bvdev;
    bdev->fn_table = &virtio_fn_table;
    bdev->module = &virtio_blk_if;

    /* 将virtio_blk_dev添加为一个io device；其IO Channel创建回调bdev_virtio_blk_ch_create_cb会申请一个
        virtio的IO环作为该IO Channel的实际对象 */
    spdk_io_device_register(bvdev, bdev_virtio_blk_ch_create_cb,
            bdev_virtio_blk_ch_destroy_cb,
            sizeof(struct bdev_virtio_blk_io_channel));

    /* 注册该bdev对象，便于后续查找 */
    rc = spdk_bdev_register(bdev);
    ...

    return 0;
}
```

<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【SPDK】五、bdev子系统](https://rootw.github.io/2018/05/SPDK-subsys-bdev/) 
