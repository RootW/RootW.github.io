---
layout: post
title: 【SPDK】三、IO流程代码解析
date: 2018-05-16 
tags: SPDK
---

&emsp;&emsp;在分析SPDK数据面代码之前，需要我们对qemu中实现的IO环以及virtio前后端驱动的实现有所了解(后续我计划出专门的博文来介绍qemu)。这里我们仍以SPDK前端配置vhost-blk，后端对接NVMe SSD为例(有关NVMe驱动涉及较多规范细节，这里也不作过于深入的讨论，感兴趣的读者可以结合NVMe规范展开阅读)进行分析。

### 总流程

&emsp;&emsp;前文在分析SPDK IO栈时已经大致分析了IO处理的调用层次，在此我们进一步打开内部实现细节，更细致地分析一下IO处理流程：

<div align="center">
<img src="/images/posts/spdk/ioanalyze.jpg" height="600" width="750">  
</div> 

&emsp;&emsp;首先，从虚拟机视角来说，它看到的是一个virtio-blk-pci设备，该pci设备内部包含一条virtio总线，其上又连接了virtio-blk设备。qemu在对虚拟机用户呈现这个virtio-blk-pci设备时，采用的具体设备类型是vhost-user-blk-pci(这是virtio-blk-pci设备的一种后端实现方式。另外两种是：vhost-blk-pci，由内核实现后端；普通virtio-blk-pci，由qemu实现后端处理)，这样便可与用户态的SPDK vhost进程建立连接。SPDK vhost进程内部对于虚拟机所见的virtio-blk-pci设备也有一个对象来表示它，这就是spdk_vhost_blk_dev。该对象指向一个bdev对象和一个io channel对象，bdev对象代表真正的后端块存储(这里对应NVMe SSD上的一个namespace)，io channel代表当前线程访问存储的独立通道(对应NVMe SSD的一个Queue Pair)。这两个对象在驱动层会进一步扩展新的成员变量，用来表示驱动层可见的一些详细信息。

&emsp;&emsp;其次，当虚拟机往IO环中放入IO请求后，便立刻被vhost进程中的某个reactor线程轮循到该请求(轮循过种中执行函数为vdev_worker)。reactor线程取出请求后，会将其映成一个任务(spdk_vhost_blk_task)。对于读写请求，会进一步走到bdev层，将任务封状成一个bdev_io对象(类似内核的bio)。bdev_io继续往驱动层递交，它会扩展为适配具体驱动的io对象，例如针对NVMe驱动，bdev_io将扩展成nvme_bdev_io对象。NVMe驱动会根据nvme_bdev_io对象中的请求内容在当前reactor线程对应的QueuePair中生成一个新的请求项，并通知NVMe控制器有新的请求产生。

&emsp;&emsp;最后，当物理NVMe控制器完成IO请求后，会往QueuePair中添加IO响应。该响应信息也会很快被reactor线程轮循到(轮循执行函数为bdev_nvme_poll)。reactor取出响应后，根据其id找到对应的nvme_bdev_io，进一步关联到对应的bdev_io，再调用bdev_io中的记录的回调函数。vhost-blk下发请求时注册的回调函数为blk_request_complete_cb，回调参数为当前的spdk_vhost_blk_task对象。在blk_request_complete_cb中会往虚拟机IO环中放入IO响应，并通过虚拟中断通知虚拟机IO完成。

### IO请求下发流程代码解析

&emsp;&emsp;vhost进程通过vdev_worker函数以轮循方式处理虚拟机下发的IO请求，调用栈如下：

```
vdev_worker()
    \-process_vq()
        |-spdk_vhost_vq_avail_ring_get()
        \-process_blk_request()
            |-blk_iovs_setup()
            \-spdk_bdev_readv()/spdk_bdev_writev()
                \-spdk_bdev_io_submit()
                    \-bdev->fn_table->submit_request()
```

&emsp;&emsp;下面我们先来分析一下vhost-blk层的具体代码实现：

```
spdk/lib/vhost/vhost-blk.c:

/* reactor线程会采用轮循方式周期性地调用vdev_worker函数来处理虚拟机下发的请求 */
static int
vdev_worker(void *arg)
{
    /* arg在注册轮循函数时指定，代表当前操作的vhost-blk对象 */
    struct spdk_vhost_blk_dev *bvdev = arg; 
    uint16_t q_idx;

    /* vhost-blk对象bvdev中含有一个抽象的spdk_vhost_dev对象，其内部记录所有vhost_dev类别对象
        均含有的公共内容，max_queues代表当前vhost_dev对象共有多少个IO环，virtqueue[]数组记录了
        所有的IO环信息 */
    for (q_idx = 0; q_idx < bvdev->vdev.max_queues; q_idx++) {
        /* 根据IO环的个数，依次处理每个环中的请求 */
        process_vq(bvdev, &bvdev->vdev.virtqueue[q_idx]);
    }

    ...

}

/* 处理IO环中的所有请求 */
static void
process_vq(struct spdk_vhost_blk_dev *bvdev, struct spdk_vhost_virtqueue *vq)
{
    struct spdk_vhost_blk_task *task;
    int rc;
    uint16_t reqs[32];
    uint16_t reqs_cnt, i;

    /* 先给出一些关于IO环的知识：
            (1) 简单来说，每个IO环分成descriptor数组、avail数组和used数组三个部分，数组元素个数均为环的最大请求个数。
            (2) descriptor数组元素代表一段虚拟机内存，每个IO请求至少包含三段，请求头部段、数据段(至少一个)和响应段。
                请求头部包含请求类型(读或写)、访问偏移，数据段代表实际的数据存放位置，响应段记录请求处理结果。一般来说，
                每个IO请求在descriptor中至少要占据三个元素；不过当配置了indirect特性后，一个IO请求只占用一项，只不过
                该项指向的内存段又是一个descriptor数组，该数组元素个数为IO请求实际所需内存段。
            (3) avail数组用来记录已下发的IO请求，数组元素内容为IO请求在descriptor数组中的下标，该下标可作为请求的id。
            (4) used数组用来记录已完成的IO响应，数组元素内容同样为IO在descritpror数组中的下标。
    */

    /* 从IO环的avail数组中中取出一批请求，将请求id放入reqs数组中；每次将环取空或者最多取32个请求 */
    reqs_cnt = spdk_vhost_vq_avail_ring_get(vq, reqs, SPDK_COUNTOF(reqs));
    ...

    /* 依次对reqs数组中的请求进行处理 */
    for (i = 0; i < reqs_cnt; i++) {
        ...
        
        /* 以请求id作为下标，找到对应的task对象。注，初始化时，会按IO环的最大请求个数来申请tasks数组 */
        task = &((struct spdk_vhost_blk_task *)vq->tasks)[reqs[i]];
        ...

        bvdev->vdev.task_cnt++; /* 作统计计数 */

        task->used = true; /* 代表tasks数组中该项正在被使用 */
        task->iovcnt = SPDK_COUNTOF(task->iovs); /* iovs数组将来会记录IO请求中数据段的内存映射信息 */
        task->status = NULL; /* 将来指向IO响应段，用来给虚拟机返回IO处理结果 */
        task->used_len = 0;

        /* 将IO环中请求的详细信息记录到task中，并递交给bdev层处理 */
        rc = process_blk_request(task, bvdev, vq);
        ...
    }
}

static int
process_blk_request(struct spdk_vhost_blk_task *task, struct spdk_vhost_blk_dev *bvdev,
struct spdk_vhost_virtqueue *vq)
{
    const struct virtio_blk_outhdr *req;
    struct iovec *iov;
    uint32_t type;
    uint32_t payload_len;
    int rc;

    /* 将IO环descriptor数组中记录的请求内存段(以gpa表示，即Guest Physical Address)映成vhost进程中的
        虚拟地址(vva, vhost virtual address)，并保存到task的iovs数组中 */
    if (blk_iovs_setup(&bvdev->vdev, vq, task->req_idx, task->iovs, &task->iovcnt, &payload_len)) {
        ...
    }

    /* 第一个请求内存段为请求头部，即struct virtio_blk_outhdr，记录请求类型、访问位置信息 */
    iov = &task->iovs[0];
    ...
    req = iov->iov_base;

    /* 最后一个请求内存段用来保存请求处理结果 */
    iov = &task->iovs[task->iovcnt - 1];
    ...
    task->status = iov->iov_base;

    /* 除去一头一尾，中间的请求内存段为数据段 */
    payload_len -= sizeof(*req) + sizeof(*task->status);
    task->iovcnt -= 2;

    type = req->type;
    
    switch (type) {
    case VIRTIO_BLK_T_IN:
    case VIRTIO_BLK_T_OUT:

        /*  对于读写请求，调用bdev读写接口，并注册请求完成后的回调函数为blk_request_complete_cb */
        if (type == VIRTIO_BLK_T_IN) {
            task->used_len = payload_len + sizeof(*task->status);
            rc = spdk_bdev_readv(bvdev->bdev_desc, bvdev->bdev_io_channel,
                    &task->iovs[1], task->iovcnt, req->sector * 512,
                    payload_len, blk_request_complete_cb, task);
        } else if (!bvdev->readonly) {
            task->used_len = sizeof(*task->status);
            rc = spdk_bdev_writev(bvdev->bdev_desc, bvdev->bdev_io_channel,
                    &task->iovs[1], task->iovcnt, req->sector * 512,
                    payload_len, blk_request_complete_cb, task);
        } else {
            SPDK_DEBUGLOG(SPDK_LOG_VHOST_BLK, "Device is in read-only mode!\n");
            rc = -1;
        }
        break;
    case VIRTIO_BLK_T_GET_ID:
        ...
        break;
    default:
        ...
        return -1;
    }   

    return 0;
}

static int
blk_iovs_setup(struct spdk_vhost_dev *vdev, struct spdk_vhost_virtqueue *vq, uint16_t req_idx,
                struct iovec *iovs, uint16_t *iovs_cnt, uint32_t *length)
{
    struct vring_desc *desc, *desc_table;
    uint16_t out_cnt = 0, cnt = 0;
    uint32_t desc_table_size, len = 0;
    int rc;

    /* 从IO环descriptor数组中获取请求对应的所有内存段信息，并映射成vva地址 */
    rc = spdk_vhost_vq_get_desc(vdev, vq, req_idx, &desc, &desc_table, &desc_table_size);
    ...

    while (1) {
        ...
        len += desc->len;

        out_cnt += spdk_vhost_vring_desc_is_wr(desc);

        rc = spdk_vhost_vring_desc_get_next(&desc, desc_table, desc_table_size);
        if (rc != 0) {
            ...
            return -1;
        } else if (desc == NULL) {
            break;
        }
    }

    ...

    *length = len;
    *iovs_cnt = cnt;
    return 0;
}

int
spdk_vhost_vq_get_desc(struct spdk_vhost_dev *vdev, struct spdk_vhost_virtqueue *virtqueue,
                    uint16_t req_idx, struct vring_desc **desc, struct vring_desc **desc_table,
                    uint32_t *desc_table_size)
{
    
    *desc = &virtqueue->vring.desc[req_idx];

    if (spdk_vhost_vring_desc_is_indirect(*desc)) {
        assert(spdk_vhost_dev_has_feature(vdev, VIRTIO_RING_F_INDIRECT_DESC));
        *desc_table_size = (*desc)->len / sizeof(**desc);
        
        /* 将IO环中记录的gpa地址转换成vhost的虚拟地址，qemu和vhost之间的内存映射关系管理我们将在管理面分析时讨论 */
        *desc_table = spdk_vhost_gpa_to_vva(vdev, (*desc)->addr, sizeof(**desc) * *desc_table_size);
        *desc = *desc_table;
        if (*desc == NULL) {
            return -1;
        }

        return 0;
    }

    *desc_table = virtqueue->vring.desc;
    *desc_table_size = virtqueue->vring.size;

    return 0;
}
```

&emsp;&emsp;接着，我们看一下bdev层对IO请求的处理，以读请求为例：

```
spdk/lib/bdev/bdev.c:

int
spdk_bdev_readv(struct spdk_bdev_desc *desc, struct spdk_io_channel *ch,
                struct iovec *iov, int iovcnt,
                uint64_t offset, uint64_t nbytes,
                spdk_bdev_io_completion_cb cb, void *cb_arg)
{
    uint64_t offset_blocks, num_blocks;

    ...
    
    /* 将字节转换成块进行实际的IO操作 */
    return spdk_bdev_readv_blocks(desc, ch, iov, iovcnt, offset_blocks, num_blocks, cb, cb_arg);
}

int spdk_bdev_readv_blocks(struct spdk_bdev_desc *desc, struct spdk_io_channel *ch,
                            struct iovec *iov, int iovcnt,
                            uint64_t offset_blocks, uint64_t num_blocks,
                            spdk_bdev_io_completion_cb cb, void *cb_arg)
{
    struct spdk_bdev *bdev = desc->bdev;
    struct spdk_bdev_io *bdev_io;
    struct spdk_bdev_channel *channel = spdk_io_channel_get_ctx(ch);

    /* io channel是一个线程强相关对象，不同的线程对应不同的channel，
        这里spdk_bdev_channel包含一个线程独立的缓存池，先从中申请bdev_io内存(免锁)，
        如果申请不到，再到全局的mempool中申请内存 */
    bdev_io = spdk_bdev_get_io(channel);
    ...

    /*  将接口参数记录到bdev_io中，并继续递交 */
    bdev_io->ch = channel;
    bdev_io->type = SPDK_BDEV_IO_TYPE_READ;
    bdev_io->u.bdev.iovs = iov;
    bdev_io->u.bdev.iovcnt = iovcnt;
    bdev_io->u.bdev.num_blocks = num_blocks;
    bdev_io->u.bdev.offset_blocks = offset_blocks;
    spdk_bdev_io_init(bdev_io, bdev, cb_arg, cb);

    spdk_bdev_io_submit(bdev_io);
    return 0;
}

static void
spdk_bdev_io_submit(struct spdk_bdev_io *bdev_io)
{
    struct spdk_bdev *bdev = bdev_io->bdev;

    if (bdev_io->ch->flags & BDEV_CH_QOS_ENABLED) { /* 开启了bdev的qos特性时走该流程 */
        ...
    } else {
        _spdk_bdev_io_submit(bdev_io); /* 直接递交 */
    }
}

static void
_spdk_bdev_io_submit(void *ctx)
{
    struct spdk_bdev_io *bdev_io = ctx;
    struct spdk_bdev *bdev = bdev_io->bdev;
    struct spdk_bdev_channel *bdev_ch = bdev_io->ch;
    struct spdk_io_channel *ch = bdev_ch->channel; /* 底层驱动对应的io channel */
    struct spdk_bdev_module_channel	*module_ch = bdev_ch->module_ch;

    bdev_io->submit_tsc = spdk_get_ticks();
    bdev_ch->io_outstanding++;
    module_ch->io_outstanding++;
    bdev_io->in_submit_request = true;
    if (spdk_likely(bdev_ch->flags == 0)) {
        if (spdk_likely(TAILQ_EMPTY(&module_ch->nomem_io))) {
            /* 不同的驱动在生成bdev对象时会注册不同的fn_table，这里将调用驱动注册的submit_request函数 */
            bdev->fn_table->submit_request(ch, bdev_io);
        } else {
            bdev_ch->io_outstanding--;
            module_ch->io_outstanding--;
            TAILQ_INSERT_TAIL(&module_ch->nomem_io, bdev_io, link);
        }
    } else if (bdev_ch->flags & BDEV_CH_RESET_IN_PROGRESS) {
        ...
    } else if (bdev_ch->flags & BDEV_CH_QOS_ENABLED) {
        ...
    } else {
        ...
    }
    bdev_io->in_submit_request = false;
}
```

&emsp;&emsp;最后，我们来看一下bdev的NVMe驱动的处理逻辑：

```
spdk/lib/bdev/bdev_nvme.c:

static const struct spdk_bdev_fn_table nvmelib_fn_table = {
    .destruct           = bdev_nvme_destruct,
    .submit_request		= bdev_nvme_submit_request,
    .io_type_supported	= bdev_nvme_io_type_supported,
    .get_io_channel		= bdev_nvme_get_io_channel,
    .dump_info_json		= bdev_nvme_dump_info_json,
    .write_config_json	= bdev_nvme_write_config_json,
    .get_spin_time		= bdev_nvme_get_spin_time,
};

static void
bdev_nvme_submit_request(struct spdk_io_channel *ch, struct spdk_bdev_io *bdev_io)
{
    int rc = _bdev_nvme_submit_request(ch, bdev_io);

    if (spdk_unlikely(rc != 0)) {
        if (rc == -ENOMEM) {
            spdk_bdev_io_complete(bdev_io, SPDK_BDEV_IO_STATUS_NOMEM);
        } else {
            spdk_bdev_io_complete(bdev_io, SPDK_BDEV_IO_STATUS_FAILED);
        }
    }
}

static int
_bdev_nvme_submit_request(struct spdk_io_channel *ch, struct spdk_bdev_io *bdev_io)
{
    /* 将ch扩展成具体的nvme_io_channel，其对应一个queue parir */
    struct nvme_io_channel *nvme_ch = spdk_io_channel_get_ctx(ch);
    if (nvme_ch->qpair == NULL) {
        /* The device is currently resetting */
        return -1;
    }

    switch (bdev_io->type) {

    /* 针对读写请求，会将bdev_io扩展成nvme_bdev_io请求后，再将请求内容填入io channel
        对应的queue pair中，并通知物理硬件处理 */
    case SPDK_BDEV_IO_TYPE_READ:
        spdk_bdev_io_get_buf(bdev_io, bdev_nvme_get_buf_cb,
                        bdev_io->u.bdev.num_blocks * bdev_io->bdev->blocklen);
        return 0;

    case SPDK_BDEV_IO_TYPE_WRITE:
        return bdev_nvme_writev((struct nvme_bdev *)bdev_io->bdev->ctxt,
                                ch,
                                (struct nvme_bdev_io *)bdev_io->driver_ctx,
                                bdev_io->u.bdev.iovs,
                                bdev_io->u.bdev.iovcnt,
                                bdev_io->u.bdev.num_blocks,
                                bdev_io->u.bdev.offset_blocks);
    ...
    default:
        return -EINVAL;
    }

    return 0;
}
```
&emsp;&emsp;详细的NVMe请求处理不在本文的讨论范围内，感兴趣的读者可以自行深入分析。

### IO响应返回流程代码解析

&emsp;&emsp;reactor线程通过bdev_nvme_poll函数获知已完成的NVMe响应，最终会调用bdev层的spdk_bdev_io_complete来处理响应：

```
spdk/lib/bdev/bdev.c:

void
spdk_bdev_io_complete(struct spdk_bdev_io *bdev_io, enum spdk_bdev_io_status status)
{
    ...
    bdev_io->status = status;

    ...
    _spdk_bdev_io_complete(bdev_io);
}

static inline void
_spdk_bdev_io_complete(void *ctx)
{
    struct spdk_bdev_io *bdev_io = ctx;

    ...

    /* 如果请求执行成功，则更新一些统计信息 */
    if (bdev_io->status == SPDK_BDEV_IO_STATUS_SUCCESS) {
        switch (bdev_io->type) {
        case SPDK_BDEV_IO_TYPE_READ:
            bdev_io->ch->stat.bytes_read += bdev_io->u.bdev.num_blocks * bdev_io->bdev->blocklen;
            bdev_io->ch->stat.num_read_ops++;
            bdev_io->ch->stat.read_latency_ticks += (spdk_get_ticks() - bdev_io->submit_tsc);
            break;
        case SPDK_BDEV_IO_TYPE_WRITE:
            bdev_io->ch->stat.bytes_written += bdev_io->u.bdev.num_blocks * bdev_io->bdev->blocklen;
            bdev_io->ch->stat.num_write_ops++;
            bdev_io->ch->stat.write_latency_ticks += (spdk_get_ticks() - bdev_io->submit_tsc);
            break;
        default:
            break;
        }
    }

    /* 调用上层注册回调，这里将回到vhost-blk的blk_request_complete_cb */
    bdev_io->cb(bdev_io, bdev_io->status == SPDK_BDEV_IO_STATUS_SUCCESS, bdev_io->caller_ctx);
}
```

```
spdk/lib/vhost/vhost_blk.c:

static void
blk_request_complete_cb(struct spdk_bdev_io *bdev_io, bool success, void *cb_arg)
{
    struct spdk_vhost_blk_task *task = cb_arg;

    spdk_bdev_free_io(bdev_io); /* 释放bdev_io */
    blk_request_finish(success, task);
}

static void
blk_request_finish(bool success, struct spdk_vhost_blk_task *task)
{
    *task->status = success ? VIRTIO_BLK_S_OK : VIRTIO_BLK_S_IOERR;

    /* 往虚拟机中放入响应并以虚拟中断方式通知虚拟机IO完成 */
    spdk_vhost_vq_used_ring_enqueue(&task->bvdev->vdev, task->vq, task->req_idx,
            task->used_len);

    /* 释放当前task，实际就是将task->used置为false */
    blk_task_finish(task);
}
```

&emsp;&emsp;至此，整个IO流程已经分析完毕，可见SPDK对IO的处理还是非常简洁的，这便是高性能的基石。

<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【SPDK】三、IO流程代码解析](https://rootw.github.io/2018/05/SPDK-ioanalyze/) 
