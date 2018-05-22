---
layout: post
title: 【SPDK】六、vhost子系统
date: 2018-05-22 
tags: SPDK
---

&emsp;&emsp;vhost子系统在SPDK中属于应用层或叫协议层，为虚拟机提供vhost-blk、vhost-scsi和vhost-nvme三种虚拟设备。这里我们以vhost-blk为分析对象，来讨论vhost子系统基本原理。

### vhost子系统初始化

&emsp;&emsp;vhost子系统的描述如下：

```
spdk/lib/event/subsystems/vhost/vhost.c:

static struct spdk_subsystem g_spdk_subsystem_vhost = {
    .name = "vhost",
    .init = spdk_vhost_subsystem_init,
    .fini = spdk_vhost_subsystem_fini,
    .config = NULL,
    .write_config_json = spdk_vhost_config_json,
};

static void
spdk_vhost_subsystem_init(void)
{
    int rc = 0;

    rc = spdk_vhost_init();

    spdk_subsystem_init_next(rc);
}
```
&emsp;&emsp;vhost子系统初始化时，会依次偿试对vhost-scsi、vhost-blk和vhost-nvme进行初始化，如果配置文件中配置了对应类型的设备，那就会完成对应设备的创建并初始化监听socket等待qemu客户端进行连接。

```
spdk/lib/vhost/vhost.c:

int
spdk_vhost_init(void)
{
    int ret;

    ...

    ret = spdk_vhost_scsi_controller_construct();
    if (ret != 0) {
        SPDK_ERRLOG("Cannot construct vhost controllers\n");
        return -1;
    }

    ret = spdk_vhost_blk_controller_construct();
    if (ret != 0) {
        SPDK_ERRLOG("Cannot construct vhost block controllers\n");
        return -1;
    }

    ret = spdk_vhost_nvme_controller_construct();
    if (ret != 0) {
        SPDK_ERRLOG("Cannot construct vhost NVMe controllers\n");
        return -1;
    }

    return 0;
}
```

### vhost-blk初始化

&emsp;&emsp;vhost-blk初始化时主要完成了两部分工作：一是vhost设备通用部分，即建立监听socket并拉起监听线程等待客户端连接；另一方面是vhost-blk特有的初始化动作，即打开bdev设备并建立联系：

```
spdk/lib/vhost/vhost_blk.c:

int
spdk_vhost_blk_construct(const char *name, const char *cpumask, const char *dev_name, bool readonly)
{
    struct spdk_vhost_blk_dev *bvdev = NULL;
    struct spdk_bdev *bdev;
    int ret = 0;

    spdk_vhost_lock();

    /* 首先通过bdev名称查找对应的bdev对象；bdev子系统在vhost子系统之前先完成初始化，正常情况下这里能找到对应的bdev */
    bdev = spdk_bdev_get_by_name(dev_name);
    ...

    bvdev = spdk_dma_zmalloc(sizeof(*bvdev), SPDK_CACHE_LINE_SIZE, NULL);
    ...

    /* 打开对应的bdev，并将句柄记录到bvdev->bdev_desc中 */
    ret = spdk_bdev_open(bdev, true, bdev_remove_cb, bvdev, &bvdev->bdev_desc);
    ...

    bvdev->bdev = bdev;
    bvdev->readonly = readonly;

    /* 完成vhost设备通用部分功能的初始化，并将该vhost设备的backend操作集合设为vhost_blk_device_backend；
        说明：不同的vhost类型实现了不同的backend，以完成不同类型特定的一些操作过程。我们在后续分析客户端连接
        操作时会深入分析backend的实现 */
    ret = spdk_vhost_dev_register(&bvdev->vdev, name, cpumask, &vhost_blk_device_backend);
    ...

    spdk_vhost_unlock();
    return ret;
}
```

&emsp;&emsp;vhost设备初始化主要提供了一个可供客户端(如qemu)连接的socket，并遵循vhost协议实现连接服务，这部分功能也是DPDK中已实现的功能，SPDK直接借用了相关代码：

```
spdk/lib/vhost/vhost.c:

int
spdk_vhost_dev_register(struct spdk_vhost_dev *vdev, const char *name, const char *mask_str,
                                            const struct spdk_vhost_dev_backend *backend)
{
    char path[PATH_MAX];
    struct stat file_stat;
    struct spdk_cpuset *cpumask;
    int rc;


    /* 将配置文件中读取的mask_str转换成位图记录到cpumask中，代表该vhost设备可以绑定的CPU核范围 */
    cpumask = spdk_cpuset_alloc();
    ...
    if (spdk_vhost_parse_core_mask(mask_str, cpumask) != 0) {
    ...
    }
    ...

    /* 生成socket文件路径名，规则是设备路径名(vhost命令启动时-S参数指定)加上vhost对象名称，
        例如 “/var/tmp/vhost.2” */
    if (snprintf(path, sizeof(path), "%s%s", dev_dirname, name) >= (int)sizeof(path)) {
        ...
    }
    ...

    /* 生成socket监听句柄 */
    if (rte_vhost_driver_register(path, 0) != 0) {
        ...
    }
    if (rte_vhost_driver_set_features(path, backend->virtio_features) ||
            rte_vhost_driver_disable_features(path, backend->disabled_features)) {
        ...
    }

    /* 注册socket连接建立后的消息处理notify_op回调 */
    if (rte_vhost_driver_callback_register(path, &g_spdk_vhost_ops) != 0) {
        ...
    }

    /* 拉起一个监听线程，开始等待客户连接请求 */
    if (spdk_call_unaffinitized(_start_rte_driver, path) == NULL) {
        ...
    }

    vdev->name = strdup(name);
    vdev->path = strdup(path);
    vdev->id = ctrlr_num++;
    vdev->vid = -1; /* 代表客户端连接对象，在客户端连接过程中生成 */
    vdev->lcore = -1; /* 代表当前vhost设备绑定到哪个核上运行，也是在客户端连接后请求处理过程中生成 */
    vdev->cpumask = cpumask;
    vdev->registered = true;
    vdev->backend = backend;

    ...

    TAILQ_INSERT_TAIL(&g_spdk_vhost_devices, vdev, tailq);

    return 0;
}
```

&emsp;&emsp;_start_rte_driver会拉起一个监听线程执行fdset_event_dispatch函数，该函数等待客户端的连接请求。当qemu向socket发起连接请求时，监听线程收到该请求并调用vhost_user_server_new_connection建立一个新的连接，然后在新的连接上等待客户端发消息。收到消息时，监听线程会调用vhost_user_read_cb函数处理消息。消息的处理代表了vhost协议的基本原理，我们将在后续独立的博文介绍。

<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【SPDK】六、vhost子系统](https://rootw.github.io/2018/05/SPDK-subsys-vhost/) 
