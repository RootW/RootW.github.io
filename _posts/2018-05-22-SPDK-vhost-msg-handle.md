---
layout: post
title: 【SPDK】七、vhost客户端连接请求处理
date: 2018-05-22 
tags: SPDK
---

&emsp;&emsp;vhost客户端连接后，将遵循vhost协议进行一系统复杂的消息传递与处理过程，最终服务端将生成一个可处理IO环中请求并返回响应的处理线程。本篇博文将分析其中最为重要两类消息的处理原理：内存映射消息和IO环信息传递消息。最后将一起来看一下vhost通用消息处理完成后，vhost-blk设备是如何完成最后的初始化动作的(其它类型的vhost设备大家可以自行阅读代码分析)。

### vhost内存映射

&emsp;&emsp;vhost的reactor线程在处理IO请求时，需要访问虚拟机的内存空间。我们知道，虚拟机可见的内存是由qemu进程分配的，通过KVM内核模块将内存映射关系记录到EPT页表中(CPU硬件提供的地址转换功能)，以此实现从GPA(Guest Physical Address)到HPA(Host Physical Address)的转换。同时qemu分配的这部分内存会映射到qemu虚拟地址空间中(Qemu Virtual Adress)，以便qemu进程中IO线程可以访问虚拟机内存。映射关系如下图所示：

<div align="center">
<img src="/images/posts/spdk/memory.jpg" height="400" width="550">  
</div> 

&emsp;&emsp;SPDK中vhost进程将取代qemu IO线程对IO进行处理，因此它也需要将虚拟机可见地址映射到自身的虚拟地址空间中(Vhost Virtual Address)，并记录VVA到HPA的映射关系，便于将HPA发送给物理存储控制器进行DMA操作。

&emsp;&emsp;vhost进程映射虚拟机地址的基本原理就是通过大页内存的mmap系统调用：

* qemu进程通过大页文件(/dev/hugepages/xxx)为虚拟机申请内存，然后将大页文件句柄传递给vhost进程；
* vhost进程接收句柄后，会识别到qemu创建的大页文件(/dev/hugepages/xxx)，然后调用mmap系统调用将该大页文件映射到自身虚拟地址空间中。

&emsp;&emsp;下面我们结合代码，再来深入理解一下内存映射过程。首先qemu连接vhost进程后，会通过发送VHOST_USER_SET_MEM_TABLE消息传递qemu内部的内存映射信息，vhost对该消息的处理过程如下：

```
spdk/lib/vhost/rte_vhost/vhost_user.c:

static int
vhost_user_set_mem_table(struct virtio_net *dev, struct VhostUserMsg *pmsg)
{
    uint32_t i;

    memcpy(&dev->mem_table, &pmsg->payload.memory, sizeof(dev->mem_table));
    memcpy(dev->mem_table_fds, pmsg->fds, sizeof(dev->mem_table_fds));
    dev->has_new_mem_table = 1;
    
    ...
    return 0;
}
```

&emsp;&emsp;从上述代码，我们可以看到这里仅是简单地将socket消息中内容复制到dev对象中。注意一点，这里的dev代表客户端对象；对象类型名为virtio_net是由于这部分代码完全借用自DPDK导致，并不是说客户端是一个virtio_net对象。

&emsp;&emsp;后续在进行gpa地址转换前，后续通过vhost_setup_mem_table完成内存映射：

```
spdk/lib/vhost/rte_vhost/vhost_user.c:

static int
vhost_setup_mem_table(struct virtio_net *dev)
{
    struct VhostUserMemory memory = dev->mem_table;
    struct rte_vhost_mem_region *reg;
    void *mmap_addr;
    uint64_t mmap_size;
    uint64_t mmap_offset;
    uint64_t alignment;
    uint32_t i;
    int fd;

    ...
    dev->mem = rte_zmalloc("vhost-mem-table", sizeof(struct rte_vhost_memory) +
            sizeof(struct rte_vhost_mem_region) * memory.nregions, 0);
    dev->mem->nregions = memory.nregions;

    for (i = 0; i < memory.nregions; i++) {
        fd  = dev->mem_table_fds[i]; /* 取出大页文件句柄，注，这里是经过内核处理后的句柄，不是qemu中的原始句柄号 */
        reg = &dev->mem->regions[i];

        reg->guest_phys_addr = memory.regions[i].guest_phys_addr; /* 虚拟机物理内存地址，gpa*/
        reg->guest_user_addr = memory.regions[i].userspace_addr;  /* qemu中的虚拟地址，qva*/
        reg->size            = memory.regions[i].memory_size;     /* 内存段大小 */
        reg->fd              = fd;

        mmap_offset = memory.regions[i].mmap_offset; /* 映射段内偏移，通常为零 */
        mmap_size   = reg->size + mmap_offset;       /* 映射段大小 */

        ...

        /* 将大页文件重新映射到当前进程中 */
        mmap_addr = mmap(NULL, mmap_size, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE, fd, 0);

        reg->mmap_addr = mmap_addr;
        reg->mmap_size = mmap_size;
        reg->host_user_addr = (uint64_t)(uintptr_t)mmap_addr + mmap_offset; /* vhost虚拟地址，vva */

        ...
    }

    return 0;
}
```

### vhost IO环信息传递

&emsp;&emsp;vhost内存映射完成后，便可进行IO环信息的传递，处理完成后使得vhost进程可以访问IO环中信息。

&emsp;&emsp;这里注意一点，vhost在处理IO环相关消息时，首先会通过vhost_user_check_and_alloc_queue_pair来创建IO环相关对象。IO环相关的消息主要有VHOST_USER_SET_VRING_NUM、VHOST_USER_SET_VRING_ADDR、VHOST_USER_SET_VRING_BASE、VHOST_USER_SET_VRING_KICK、VHOST_USER_SET_VRING_CALL，这里我们重点分析一下VHOST_USER_SET_VRING_ADDR消息的处理：

```
spdk/lib/vhost/rte_vhost/vhost_user.c:

static int
vhost_user_set_vring_addr(struct virtio_net *dev, VhostUserMsg *msg)
{
    struct vhost_virtqueue *vq;
    uint64_t len;

    /* 如果还未完成vhost内存的映射，则先进行内存映射，可参考前文分析 */
    if (dev->has_new_mem_table) {
        vhost_setup_mem_table(dev);
        dev->has_new_mem_table = 0;
    }
    ...

    /* 根据消息中的索引找到对应的vq对象 */
    vq = dev->virtqueue[msg->payload.addr.index];

    /* The addresses are converted from QEMU virtual to Vhost virtual. */
    len = sizeof(struct vring_desc) * vq->size;
    /* 将消息中包含的desc数组的qva地址转换成vva地址，便于vhost线程后续访问IO环中desc数组中内容 */
    vq->desc = (struct vring_desc *)(uintptr_t)qva_to_vva(dev, msg->payload.addr.desc_user_addr, &len);

    dev = numa_realloc(dev, msg->payload.addr.index);
    vq = dev->virtqueue[msg->payload.addr.index];

    /* 同理将avail数组的qva地址转换成vva地址 */
    len = sizeof(struct vring_avail) + sizeof(uint16_t) * vq->size;
    vq->avail = (struct vring_avail *)(uintptr_t)qva_to_vva(dev, msg->payload.addr.avail_user_addr, &len);
    
    /* 同理将used数组的qva地址转换成vva地址 */
    len = sizeof(struct vring_used) + sizeof(struct vring_used_elem) * vq->size;
    vq->used = (struct vring_used *)(uintptr_t)qva_to_vva(dev, msg->payload.addr.used_user_addr, &len);

    ...
    return 0;
}

```

### vhost-blk回调处理

&emsp;&emsp;vhost设备完成内存映射及IO环信息传递动作后，就进行不同vhost设备特有的初始化动作：

```
spdk/lib/vhost/rte_vhost/vhost_user.c:

int
vhost_user_msg_handler(int vid, int fd)
{

    /* 从socket句柄中读取消息 */
    ret = read_vhost_message(fd, &msg);
    ...

    /* 如果消息中涉及IO环则先创建IO环对象 */
    ret = vhost_user_check_and_alloc_queue_pair(dev, &msg);

    /* 根据不同的消息类型进行处理 */
    switch (msg.request) {
    case VHOST_USER_GET_CONFIG:
    ...
    }

    if (!(dev->flags & VIRTIO_DEV_RUNNING) && virtio_is_ready(dev)) {
        dev->flags |= VIRTIO_DEV_READY;

        if (!(dev->flags & VIRTIO_DEV_RUNNING)) {

            /* 通过notify_ops回调设备相关的初始化函数 */
            if (dev->notify_ops->new_device(dev->vid) == 0)
                dev->flags |= VIRTIO_DEV_RUNNING;
        }
    }

    return 0;
}
```

&emsp;&emsp;g_spdk_vhost_ops的new_device函数指向start_device，这里仍是vhost设备通用的初始化逻辑：

```
spdk/lib/vhost/vhost.c:

static int
start_device(int vid)
{
    struct spdk_vhost_dev *vdev;
    int rc = -1;
    uint16_t i;

    /* 根据客户端vid找到对应的vhost_dev设备 */
    vdev = spdk_vhost_dev_find_by_vid(vid);

    /* 将客户端对象(virtio_net)中记录的IO环信息同步一份到vhost_dev中，后续IO处理时主要操作vhost_dev对象 */
    vdev->max_queues = 0;
    memset(vdev->virtqueue, 0, sizeof(vdev->virtqueue));
    for (i = 0; i < SPDK_VHOST_MAX_VQUEUES; i++) {
        if (rte_vhost_get_vhost_vring(vid, i, &vdev->virtqueue[i].vring)) {
            continue;
        }

        if (vdev->virtqueue[i].vring.desc == NULL ||
                vdev->virtqueue[i].vring.size == 0) {
            continue;
        }

        /* Disable notifications. */
        if (rte_vhost_enable_guest_notification(vid, i, 0) != 0) {
            SPDK_ERRLOG("vhost device %d: Failed to disable guest notification on queue %"PRIu16"\n", vid, i);
            goto out;
        }

        vdev->max_queues = i + 1;
    }

    /* 同理，将客户端对象中的内存映射表同步一份到vhost_dev中 */
    if (rte_vhost_get_mem_table(vid, &vdev->mem) != 0) {
        
    }

    /* 为vhost_dev对象分配一个运行核 */
    vdev->lcore = spdk_vhost_allocate_reactor(vdev->cpumask);

    /* 记录该vdev对象内存表中虚拟地址到物理地址的映射关系，后续操作物理DMA时可用 */
    spdk_vhost_dev_mem_register(vdev);

    /* 向vhost_dev对象的运行核发送一个事件，使该核上的reactor线程可以执行backend的start_device函数 */
    rc = spdk_vhost_event_send(vdev, vdev->backend->start_device, 3, "start device");
    ...

    return rc;
}
```

&emsp;&emsp;vhost_dev的运行核上的reactor线程会执行backend的start_device，即spdk_vhost_blk_start：

```
spdk/lib/vhost/vhost_blk.c:

static int
spdk_vhost_blk_start(struct spdk_vhost_dev *vdev, void *event_ctx)
{
    struct spdk_vhost_blk_dev *bvdev;
    int i, rc = 0;

    bvdev = to_blk_dev(vdev);
    ...

    /* 为vhost设备中的每个队列分配task数组，task与队列中元素个数相同，一一对应 */
    rc = alloc_task_pool(bvdev);
    ...

    if (bvdev->bdev) {
        /* 为vhost_blk对应申请IO Channel，此时已确定执行线程上下文 */
        bvdev->bdev_io_channel = spdk_bdev_get_io_channel(bvdev->bdev_desc);
        ...
    }

    /* 在当前reactor线程中添加一个poller，用来处理IO环中的所有请求 */
    bvdev->requestq_poller = spdk_poller_register(bvdev->bdev ? vdev_worker : no_bdev_vdev_worker, bvdev, 0);
    ...
    return rc;
}
```

&emsp;&emsp;至此，SPDK中vhost进程的初始化流程已介绍完毕，过程非常漫长，大家可以在对数据面的处理流程有一定的熟悉之后再来阅读分析这部分代码，这样可以理解得更深刻。

<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【SPDK】七、vhost客户端连接请求处理](https://rootw.github.io/2018/05/SPDK-vhost-msg-handle/) 
