---
layout: post
title: Rados Block Device之二－客户端内核RBD驱动分析
date: 2018-01-05 
tags: ceph
---

&emsp;&emsp;下面我们通过分析客户端内核驱动，而将服务端(Monitor，OSD)当作黑盒来理解整个ceph系统原理。如果按软件栈分层，linux内核中的RBD驱动涉及上中下两层：上层是在内核块层中的RBD驱动(rbd.ko)；中层是在内核网络层中的libceph驱动(libceph.ko)；下层内核网络socket通信。rbd.ko为客户端提供可访问的块设备，libceph.ko为rbd.ko提供对象存储接口，socket则提供最底层的网络通信，软件栈及其核心对象概览如下：

<div align="center">
<img src="/images/posts/i440fx/rbd_4.jpg" height="400" width="600">  
</div> 

### 1. RBD块设备创建

&emsp;&emsp;回顾开篇介绍RBD块设备的使用方法，分为创建、映射和使用三步。创建过程主要是用户态rbd工具与rados集群通信完成的，不涉及内核RBD驱动；但问题是rados集群中的OSD提供的是对象服务(我们可以把对象简单理解成key/value对)，如何来表达一个RBD块设备？

&emsp;&emsp;我们通过rados ls命令来查看一下wbpool中创建了一个名为wb的RBD设备后都生成了哪些对象：

> [root@ceph-client]# **rados ls \-\-pool** wbpool  
> rbd_header.371e643c9869  
> rbd_directory    
> rbd_object_map.371e643c9869  
> rbd_id.wb  
> rbd_info  

&emsp;&emsp;这些对象里到底包含了哪些信息呢？我们可以用rados get命令获取对象内容到文件中并打开文件进行查看。先看看rbd_id.wb对象内容：

> [root@ceph-client]# **rados get** rbd_id.wb ./result.txt **\-\-pool** wbpool; **hexdump -Cv** ./result.txt  
> 00000000  0c 00 00 00 33 37 31 65  36 34 33 63 39 38 36 39 |....371e643c9869|  
> 00000010  

&emsp;&emsp;从文件内容看，rbd_id.wb中保存的主要是个id信息，为"371e643c9869"(前四个字节代表什么意思？通过小端法表示对象实际内容的字节长度，例如这里是0x0000000C，即12个字节)。知道了wb块设备的id，我们再看来看看rbd_head.371e643c9869对象中的内容：

> [root@ceph-client]# rados get rbd_header.371e643c9869 ./result.txt --pool wbpool; hexdump -Cv ./result.txt  
>  

&emsp;&emsp;rbd_header.371e643c9869对象内容为空，怎么回事呢？我们再来看看该对象相关的key/value omap值(可以用来描述对象的元数据信息)：

> [root@ceph-client]# **rados listomapvals** rbd_header.371e643c9869 **\-\-pool** wbpool  
> create_timestamp  
> value (8 bytes) :  
> 00000000  63 38 26 5a 84 c1 71 08 |c8&Z..q.|  
> 00000008  
>  
> features  
> value (8 bytes) :  
> 00000000  01 00 00 00 00 00 00 00 |........|  
> 00000008  
>  
> flags  
> value (8 bytes) :  
> 00000000  00 00 00 00 00 00 00 00 |........|  
> 00000008  
>  
> object_prefix  
> value (25 bytes) :  
> 00000000  15 00 00 00 72 62 64 5f  64 61 74 61 2e 33 37 31 |....rbd_data.371|  
> 00000010  65 36 34 33 63 39 38 36  39 |e643c9869|  
> 00000019  
> 
> order 
> value (1 bytes) :  
> 00000000  16                                                |.|  
> 00000001  
>  
> size  
> value (8 bytes) :  
> 00000000  00 00 00 40 00 00 00 00 |...@....|  
> 00000008  
>  
> snap_seq  
> value (8 bytes) :  
> 00000000  00 00 00 00 00 00 00 00 |........|  
> 00000008  

&emsp;&emsp;这样我们看到rbd_header.371e643c9869对象的omap中保存与RBD相关的元数据信息，如RBD数据存放对象前缀为"rbd_data.371e643c9869"；order为0x16，表示以4M单位来划分RBD块设备，每4M对应一个对象，对象名为"rbd_data.371e643c9869.偏移"；size为0x40000000，即1G。

&emsp;&emsp;最后我们来看看另外两个与具体rbd设备无关的对象rbd_info和rbd_directory：

> [root@ceph-client]# **rados get** rbd_info ./result.txt **\-\-pool** wbpool; **hexdump -Cv** ./result.txt  
> 00000000  6f 76 65 72 77 72 69 74  65 20 76 61 6c 69 64 61 |overwrite valida|  
> 00000010  74 65 64                                          |ted|  
> 00000013  
>  
> [root@ceph-client]# **rados get** rbd_directory ./result.txt **\-\-pool** wbpool; **hexdump -Cv** ./result.txt  
> [root@ceph-client]# **rados listomapvals** rbd_directory **\-\-pool** wbpool  
> id_371e643c9869  
> value (6 bytes) :  
> 00000000  02 00 00 00 77 62                                 |....wb|  
> 00000006  
>  
> name_wb  
> value (16 bytes) :  
> 00000000  0c 00 00 00 33 37 31 65  36 34 33 63 39 38 36 39 |....371e643c9869|  
> 00000010  

&emsp;&emsp;从结果我们看出rbd_info中只是保存了一个提示字符串"overwirte validated"。rbd_directory对象的map中保存了所有RBD块设备的名字和id的对应关系，可实现双向查找。

&emsp;&emsp;综上所述，ceph中也是利用了对象存储的方式来保存RBD信息："rbd_id.块设备名"对象中保存对应块设备的id；"rbd_header.块设备id"的omap中保存了块设备的各种元数据信息(这里没有直接保存的对象内容中，通过omap更易解析)；"rbd_data.块设备id.偏移"对象用来保存块设备对应偏移处的数据；rbd_directory用来保存pool中所有的RBD设备的名称和id。

### 2. RBD块设备映射流程分析

&emsp;&emsp;rbd工具通过map子命令映射产生一个可使用的内核块设备，其本质原理是由rbd工具往sysfs("/sys/bus/rbd/add")内核文件接口中写入待创建块设备的信息(例如前文实例写入的内容为"9.22.115.154:6789 name=amdin,key=client, wbpool wb -"，其中9.22.115.154为主monitor的IP)，内核在处理这个写入请求时会调用由RBD驱动事先注册好的处理函数来生成一个内核块设备对象。

&emsp;&emsp;我们先来看看rbd模块加载初始化时完成了哪些初始化动作：

```
linux/drivers/block/rbd.c:

static int __init rbd_init(void)
{
    int rc;

    /*首先检查rbd驱动依赖的底层libceph驱动的兼容性*/
    if (!libceph_compatible(NULL)) {
        rbd_warn(NULL, "libceph incompatibility (quitting)");

        return -EINVAL;
    }

    /*初始化rbd驱动中使用的内存分配器*/
    rc = rbd_slab_init();
    if (rc)
        return rc;

    /*sysfs文件接口初始化，为用户态rbd工具暴露访问接口*/
    rc = rbd_sysfs_init();
    if (rc)
        rbd_slab_exit();
    else
    pr_info("loaded " RBD_DRV_NAME_LONG "\n");

    return rc;
}

module_init(rbd_init);
```

```
linux/drivers/block/rbd.c:

/*
 * create control files in sysfs
 * /sys/bus/rbd/...
 */
static int rbd_sysfs_init(void)
{
    int ret;

    ret = device_register(&rbd_root_dev);
    if (ret < 0)
        return ret;

    /*根据rbd_bus_type定义的信息在/sys/bus/下生成子目录*/
    ret = bus_register(&rbd_bus_type);
    if (ret < 0)
        device_unregister(&rbd_root_dev);

    return ret;
}

static struct bus_attribute rbd_bus_attrs[] = {
    __ATTR(add, S_IWUSR, NULL, rbd_add), /*对rbd目录下的add文件进行写操作时将调用rbd_add*/
    __ATTR(remove, S_IWUSR, NULL, rbd_remove), /*对rbd目录下的remove文件进行写操作时将调用rbd_remove*/
    __ATTR_NULL
};

static struct bus_type rbd_bus_type = {
    .name		= "rbd", /* /sys/bus/成生的子目录名 */
    .bus_attrs	= rbd_bus_attrs,
};

```

&emsp;&emsp;下面我们顺着本文开头给出的软件栈从上至下(rbd->libceph)深入分析rbd_add函数，看看内核RBD驱动是如何进行块设备的生成的：

```
linux/drivers/block/rbd.c:

/*用户态工具往/sys/bus/rbd/add中写入RBD设备信息，内核在处理该写入请求时就会
  调用这里的rbd_add，其中bus就是/sys/bus/rbd对象，buf中存放用户态工具写入的
  字符串，如"9.22.115.154:6789 name=amdin,key=client, wbpool wb -"，
  count为写入的字符串长度*/

static ssize_t rbd_add(struct bus_type *bus,
                const char *buf, size_t count)
{
    struct rbd_device *rbd_dev = NULL;
    struct ceph_options *ceph_opts = NULL;
    struct rbd_options *rbd_opts = NULL;
    struct rbd_spec *spec = NULL;
    struct rbd_client *rbdc;
    struct ceph_osd_client *osdc;
    bool read_only;
    int rc = -ENOMEM;

    ...

    /*首先对用户态写入的字符串进行解析，结果返回到ceph_opts，rbd_opt和spec中：
      (1)ceph_opts中保存ceph集群相关信息，针对前文示例，ceph_opts.mon_addr中将保存
         "9.22.115.154:6789"；ceph_opts.name为"admin"，代表访问ceph集群的角色名称；
         ceph_opts.key对应系统配置中的"ceph.client.admin.keyring"，为登陆密钥。
      (2)rbd_opt中包含当前rbd设备是否只读，针对前文示例，rbd_opt.read_only为false。
      (3)spec中包含当前rbd设备的详细描述信息，针对示例，spec.pool_name为"wb_pool"，
         spec.image_name为"wb"，spec.snap_name为"-"，表示无快照*/

    /* parse add command */
    rc = rbd_add_parse_args(buf, &ceph_opts, &rbd_opts, &spec);
    if (rc < 0)
        goto err_out_module;
    read_only = rbd_opts->read_only;
    kfree(rbd_opts);
    rbd_opts = NULL;	/* done with this */

    /*接着根据ceph集群配置ceph_opts在当前客户端查找是否已生成rbd_client(对应一个ceph_client，
      内部包含mon_client和osd_client)；如果未找到则生成一个新的rbd_client，并建立会话(完成认证、获
      取monmap和osdmap等一系统初始化动作)*/
    rbdc = rbd_get_client(ceph_opts);
    if (IS_ERR(rbdc)) {
        rc = PTR_ERR(rbdc);
        goto err_out_args;
    }

    /*从ceph集群返回的osdmap中获取当前pool对应的id*/
    /* pick the pool */
    osdc = &rbdc->client->osdc;
    rc = ceph_pg_poolid_by_name(osdc->osdmap, spec->pool_name);
    if (rc < 0)
        goto err_out_client;
    spec->pool_id = (u64)rc;

    ...

    /*创建内核中的rbd_dev对象*/
    rbd_dev = rbd_dev_create(rbdc, spec);
    if (!rbd_dev)
        goto err_out_client;
    rbdc = NULL;		/* rbd_dev now owns this */
    spec = NULL;		/* rbd_dev now owns this */

    /*从ceph集群中查询当前rbd设备(image)的元数据信息，即从rbd_id.[rbd设备名]中获知image id，并从
      rbd_header.[image id]的omap中获得设备大小，分片大小等信息*/
    rc = rbd_dev_image_probe(rbd_dev, true);
    if (rc < 0)
        goto err_out_rbd_dev;

    ...

    /*根据ceph集群返回的rbd信息，完成对rbd_dev的设置，并通过内核块层接口生成新的块设备对象*/
    rc = rbd_dev_device_setup(rbd_dev);
    if (rc) {
        rbd_dev_image_release(rbd_dev);
        goto err_out_module;
    }

    return count;

    ...
}

```

#### **2.1. rbd_get_client**

&emsp;&emsp;针对同一个rados集群，每个客户端节点都会在内核中生成一个rbd_client对象，用来和该rados集群进行会话；在一个客户端节点上，如果针对同一个rados集群创建了多个RBD设备，那么这些RBD设备会共用同一个rbd_client。下面我们看看rbd_get_client的内部实现：

```
linux/drivers/block/rbd.c:

/*
 * Get a ceph client with specific addr and configuration, if one does
 * not exist create it.  Either way, ceph_opts is consumed by this
 * function.
 */
static struct rbd_client *rbd_get_client(struct ceph_options *ceph_opts)
{
    struct rbd_client *rbdc;

    rbdc = rbd_client_find(ceph_opts); /*在全局链表rbd_client_list中根据ceph_opts的集群信息查找rbd_client*/
    if (rbdc)	/* using an existing client */
        ceph_destroy_options(ceph_opts); /*如果找到相同的rbd_client(对应同一个rados集群)，则复用该对象，同时销毁ceph_opts*/
    else
        rbdc = rbd_client_create(ceph_opts); /*如果没有找到相同的rbd_client，则新建一个并添加到rbd_client_list中*/

    return rbdc;
}
```

&emsp;&emsp;接着看rbd_client_create：

```
linux/drivers/block/rbd.c:

/*rbd_client代表rbd客户端实例，多个rbd设备可共用一个客户端实例；其内部主要包含一个ceph_client对象*/
/*
 * an instance of the client.  multiple devices may share an rbd client.
 */
struct rbd_client {
    struct ceph_client	*client;
    struct kref		kref;
    struct list_head	node;
};

/*
 * Initialize an rbd client instance.  Success or not, this function
 * consumes ceph_opts.
 */
static struct rbd_client *rbd_client_create(struct ceph_options *ceph_opts)
{
    struct rbd_client *rbdc;
    int ret = -ENOMEM;

    rbdc = kmalloc(sizeof(struct rbd_client), GFP_KERNEL);
    ...

    /*调用libceph.ko中的函数创建ceph_client对象*/
    rbdc->client = ceph_create_client(ceph_opts, rbdc, 0, 0);
    ...

    /*调用libceph.ko中的函数打开ceph_client对象的会话，基于该会话可以和rados集群(monitor or OSD)进行通信*/
    ret = ceph_open_session(rbdc->client);
    ...

    spin_lock(&rbd_client_list_lock);
    list_add_tail(&rbdc->node, &rbd_client_list);
    spin_unlock(&rbd_client_list_lock);

    ...

    return rbdc;

    ...
}
```

&emsp;&emsp;下面我们再深入一层，看看libceph中ceph_client的相关实现，但不会深入到messenger模块中(复杂性较高)，我们在2.1.3节会总结一下对messenger模块的使用方法(API)。对于messenger的分析我们将放到本篇博文最后一节。

##### **2.1.1. rbd_get_client -> ceph_create_client**

```
linux/include/linux/ceph/libceph.h:

/*
 * per client state
 *
 * possibly shared by multiple mount points, if they are
 * mounting the same ceph filesystem/cluster.
 */
struct ceph_client {
    ...

    /*每个ceph_client包含一个messenger实例、一个mon_client实例和一个osd_client实例：
      (1)messenger实例描述了与网络通信相关的信息，属于messenger模块，是ceph客户端基于socket封装的网络服务层；
      (2)mon_client实例是专门用来与monitor通信的客户端；
      (3)osd_client实例是专门用来与所有osd通信的客户端。*/
    struct ceph_messenger msgr;   /* messenger instance */
    struct ceph_mon_client monc;
    struct ceph_osd_client osdc;

    ...
};
```

```
linux/net/ceph/ceph_common.c:

/*ceph_create_client创建ceph_client对象并对其内部的messenger、mon_client、osd_client进行初始化*/
/*
 * create a fresh client instance
 */
struct ceph_client *ceph_create_client(struct ceph_options *opt, void *private,
        unsigned int supported_features, unsigned int required_features)
{
    struct ceph_client *client;
    struct ceph_entity_addr *myaddr = NULL;
    int err = -ENOMEM;

    client = kzalloc(sizeof(*client), GFP_KERNEL);
    if (client == NULL)
        return ERR_PTR(-ENOMEM);

    client->private = private;
    client->options = opt;

    ...

    /* msgr */
    if (ceph_test_opt(client, MYIP))
        myaddr = &client->options->my_addr;
    ceph_messenger_init(&client->msgr, myaddr, client->supported_features,
        client->required_features, ceph_test_opt(client, NOCRC));

    /* subsystems */
    err = ceph_monc_init(&client->monc, client);
    if (err < 0)
        goto fail;
    err = ceph_osdc_init(&client->osdc, client);
    if (err < 0)
        goto fail_monc;

    return client;
    ...
}
```

&emsp;&emsp;接着我们再深入看看messenger、mon_client、osd_client的初始化：

```
linux/include/linux/ceph/messenger.h:

struct ceph_messenger {
    struct ceph_entity_inst inst;    /* my name+address */
    struct ceph_entity_addr my_enc_addr;

    atomic_t stopping;
    bool nocrc;

    /*
     * the global_seq counts connections i (attempt to) initiate
     * in order to disambiguate certain connect race conditions.
     */
    u32 global_seq;
    spinlock_t global_seq_lock;

    u32 supported_features;
    u32 required_features;
};
```
```
linux/net/ceph/messenger.c:

/*
 * initialize a new messenger instance
 */
void ceph_messenger_init(struct ceph_messenger *msgr,
            struct ceph_entity_addr *myaddr,
            u32 supported_features,
            u32 required_features,
            bool nocrc)
{
    msgr->supported_features = supported_features;
    msgr->required_features = required_features;

    spin_lock_init(&msgr->global_seq_lock);

    if (myaddr)
        msgr->inst.addr = *myaddr;

    /* select a random nonce */
    msgr->inst.addr.type = 0;
    get_random_bytes(&msgr->inst.addr.nonce, sizeof(msgr->inst.addr.nonce));
    encode_my_addr(msgr);
    msgr->nocrc = nocrc;

    atomic_set(&msgr->stopping, 0);

    dout("%s %p\n", __func__, msgr);
}
```

```
linux/include/linux/ceph/mon_client.h:

struct ceph_mon_client {
    struct ceph_client *client;
    struct ceph_monmap *monmap;

    struct mutex mutex;
    struct delayed_work delayed_work;

    struct ceph_auth_client *auth;
    struct ceph_msg *m_auth, *m_auth_reply, *m_subscribe, *m_subscribe_ack;
    int pending_auth;

    bool hunting;
    int cur_mon;                       /* last monitor i contacted */
    unsigned long sub_sent, sub_renew_after;
    struct ceph_connection con;

    /* pending generic requests */
    struct rb_root generic_request_tree;
    int num_generic_requests;
    u64 last_tid;

    /* mds/osd map */
    int want_mdsmap;
    int want_next_osdmap; /* 1 = want, 2 = want+asked */
    u32 have_osdmap, have_mdsmap;
};

```
```
linux/net/ceph/mon_client.c:

int ceph_monc_init(struct ceph_mon_client *monc, struct ceph_client *cl)
{
    int err = 0;

    dout("init\n");
    memset(monc, 0, sizeof(*monc));
    monc->client = cl;
    monc->monmap = NULL;
    mutex_init(&monc->mutex);

    err = build_initial_monmap(monc);
    if (err)
        goto out;

    /* connection */
    /* authentication */
    monc->auth = ceph_auth_init(cl->options->name, cl->options->key);
    ...
    monc->auth->want_keys =
        CEPH_ENTITY_TYPE_AUTH | CEPH_ENTITY_TYPE_MON |
        CEPH_ENTITY_TYPE_OSD | CEPH_ENTITY_TYPE_MDS;

    /* msgs */
    err = -ENOMEM;
    monc->m_subscribe_ack = ceph_msg_new(CEPH_MSG_MON_SUBSCRIBE_ACK,
        sizeof(struct ceph_mon_subscribe_ack), GFP_NOFS, true);
    ...
    monc->m_subscribe = ceph_msg_new(CEPH_MSG_MON_SUBSCRIBE, 96, GFP_NOFS, true);
    ...
    monc->m_auth_reply = ceph_msg_new(CEPH_MSG_AUTH_REPLY, 4096, GFP_NOFS, true);
    ...
    monc->m_auth = ceph_msg_new(CEPH_MSG_AUTH, 4096, GFP_NOFS, true);
    monc->pending_auth = 0;
    ...

    /*mon_client通过底层messenger模块中的connection对象进行网络通信，这里对mon_client使用的connection进行
      初始化，并指定其网络层回调函数集为mon_con_ops，用来处理消息回调、网络连接故障等*/
    ceph_con_init(&monc->con, monc, &mon_con_ops, &monc->client->msgr);

    monc->cur_mon = -1;
    monc->hunting = true;
    monc->sub_renew_after = jiffies;
    monc->sub_sent = 0;

    INIT_DELAYED_WORK(&monc->delayed_work, delayed_work);
    monc->generic_request_tree = RB_ROOT;
    monc->num_generic_requests = 0;
    monc->last_tid = 0;

    monc->have_mdsmap = 0;
    monc->have_osdmap = 0;
    monc->want_next_osdmap = 1;
    return 0;

    ...
}
```

```
linux/include/linux/ceph/osd_client.h:

struct ceph_osd_client {
    struct ceph_client     *client;

    struct ceph_osdmap     *osdmap;       /* current map */
    struct rw_semaphore    map_sem;
    struct completion      map_waiters;
    u64                    last_requested_map;

    struct mutex           request_mutex;
    struct rb_root         osds;          /* osds */
    struct list_head       osd_lru;       /* idle osds */
    u64                    timeout_tid;   /* tid of timeout triggering rq */
    u64                    last_tid;      /* tid of last request */
    struct rb_root         requests;      /* pending requests */
    struct list_head       req_lru;	      /* in-flight lru */
    struct list_head       req_unsent;    /* unsent/need-resend queue */
    struct list_head       req_notarget;  /* map to no osd */
    struct list_head       req_linger;    /* lingering requests */
    int                    num_requests;
    struct delayed_work    timeout_work;
    struct delayed_work    osds_timeout_work;

    mempool_t              *req_mempool;

    struct ceph_msgpool	msgpool_op;
    struct ceph_msgpool	msgpool_op_reply;

    spinlock_t		event_lock;
    struct rb_root		event_tree;
    u64			event_count;

    struct workqueue_struct	*notify_wq;
};
```

```
linux/net/ceph/osd_client.c:

int ceph_osdc_init(struct ceph_osd_client *osdc, struct ceph_client *client)
{
    int err;

    dout("init\n");
    osdc->client = client;
    osdc->osdmap = NULL;
    init_rwsem(&osdc->map_sem);
    init_completion(&osdc->map_waiters);
    osdc->last_requested_map = 0;
    mutex_init(&osdc->request_mutex);
    osdc->last_tid = 0;
    osdc->osds = RB_ROOT;
    INIT_LIST_HEAD(&osdc->osd_lru);
    osdc->requests = RB_ROOT;
    INIT_LIST_HEAD(&osdc->req_lru);
    INIT_LIST_HEAD(&osdc->req_unsent);
    INIT_LIST_HEAD(&osdc->req_notarget);
    INIT_LIST_HEAD(&osdc->req_linger);
    osdc->num_requests = 0;
    INIT_DELAYED_WORK(&osdc->timeout_work, handle_timeout);
    INIT_DELAYED_WORK(&osdc->osds_timeout_work, handle_osds_timeout);
    spin_lock_init(&osdc->event_lock);
    osdc->event_tree = RB_ROOT;
    osdc->event_count = 0;

    schedule_delayed_work(&osdc->osds_timeout_work,
        round_jiffies_relative(osdc->client->options->osd_idle_ttl * HZ));

    err = -ENOMEM;
    osdc->req_mempool = mempool_create_kmalloc_pool(10, sizeof(struct ceph_osd_request));
    if (!osdc->req_mempool)
        goto out;

    err = ceph_msgpool_init(&osdc->msgpool_op, CEPH_MSG_OSD_OP,
            OSD_OP_FRONT_LEN, 10, true, "osd_op");
    if (err < 0)
        goto out_mempool;
    err = ceph_msgpool_init(&osdc->msgpool_op_reply, CEPH_MSG_OSD_OPREPLY,
        OSD_OPREPLY_FRONT_LEN, 10, true, "osd_op_reply");
    if (err < 0)
        goto out_msgpool;

    err = -ENOMEM;
    osdc->notify_wq = create_singlethread_workqueue("ceph-watch-notify");
    if (!osdc->notify_wq)
        goto out_msgpool;
    return 0;

out_msgpool:
    ceph_msgpool_destroy(&osdc->msgpool_op);
out_mempool:
    mempool_destroy(osdc->req_mempool);
out:
    return err;
}
```

##### **2.1.2. rbd_get_client -> ceph_open_session**

&emsp;&emsp;在完成ceph_client的初始化动作后，下一步是打开会话：

```
linux/net/ceph/ceph_common.c:

int ceph_open_session(struct ceph_client *client)
{
    ...

    ret = __ceph_open_session(client, started);

    ...
    return ret;
}

/*
 * mount: join the ceph cluster, and open root directory.
 */
int __ceph_open_session(struct ceph_client *client, unsigned long started)
{
    int err;
    /*timeout表示打开会话的超时时间，默认为60秒*/
    unsigned long timeout = client->options->mount_timeout * HZ;

    /*ceph_client内部是通过mon_client来打开会话，如果mon_client成功打开会话，
      它会成功获得rados集群的mon_map(描述所有monitors信息)和osd_map(描述所有的osd信息)*/
    /* open session, and wait for mon and osd maps */
    err = ceph_monc_open_session(&client->monc);
    if (err < 0)
        return err;

    while (!have_mon_and_osd_map(client)) { /*如果没有获得mon_map和osd_map，则等待直到超时*/
        err = -EIO;
        if (timeout && time_after_eq(jiffies, started + timeout)) /*超时退出*/
            return err;

        /* wait */
        dout("mount waiting for mon_map\n");
        /*下面，当前进程(rbd工具)将进入睡眠状态，直到内核接收到mon_map和osd_map时被重新唤醒，
          或者出现认证错时也将被唤醒*/
        err = wait_event_interruptible_timeout(client->auth_wq,
            have_mon_and_osd_map(client) || (client->auth_err < 0),
            timeout);
        if (err == -EINTR || err == -ERESTARTSYS)
            return err;
        if (client->auth_err < 0)
            return client->auth_err;
    }

    return 0;
}
```

```
linux/net/ceph/mon_client.c:

int ceph_monc_open_session(struct ceph_mon_client *monc)
{
    mutex_lock(&monc->mutex);
    __open_session(monc);
    __schedule_delayed(monc);
    mutex_unlock(&monc->mutex);
    return 0;
}

/*
 * Open a session with a (new) monitor.
 */
static int __open_session(struct ceph_mon_client *monc)
{
    char r;
    int ret;

    if (monc->cur_mon < 0) {/*初始时cur_mon为－1，表示没有和任何monitor建立连接*/

        /*通过随机数r，从初始化时生成的mon_map中随机选一个进行会话连接的monitor*/
        get_random_bytes(&r, 1);
        monc->cur_mon = r % monc->monmap->num_mon;
        monc->sub_sent = 0;
        monc->sub_renew_after = jiffies;  /* i.e., expired */
        monc->want_next_osdmap = !!monc->want_next_osdmap;

        /*与选定的monitor建立网络连接，这里使用的messenger模块提供的接口*/
        ceph_con_open(&monc->con,
            CEPH_ENTITY_TYPE_MON, monc->cur_mon,
            &monc->monmap->mon_inst[monc->cur_mon].addr);

        /*初始化发送给monitor的首个hello消息*/
        /* initiatiate authentication handshake */
        ret = ceph_auth_build_hello(monc->auth,
                monc->m_auth->front.iov_base,
                monc->m_auth->front_alloc_len);

        /*这里通过messenger模块的ceph_con_send接口将消息发送给monitor*/
        __send_prepared_auth_request(monc, ret);
    } else {
        dout("open_session mon%d already open\n", monc->cur_mon);
    }
    return 0;
}
```

##### **2.1.3. messenger模块使用方法小结**

&emsp;&emsp;通过前文对mon_client的分析，我们来总结一下mon_client是如何使用底层的messenger模块的：

>* (1) 通过ceph_messenger_init在ceph_client中初始化一个messenger对象，定义全局网络通信信息；该对象被mon_client和osd_client共享；
>* (2) ceph_con_init(&monc->con, monc, &mon_con_ops, &monc->client->msgr)，网络连接对象初始化，并指明该连接收到消息时的回调处理函数(消息将在内核工作队列上下文被处理)； 
>* (3) ceph_con_open打开与选定monitor的网络连接；并拉起一个与当前连接关联的工作任务(connection worker)，该任务负责该连接上网络消息的收发；
>* (4) 发送消息时，先使用ceph_msg_new进行消息的内存分配，再填入消息内容，然后使用ceph_con_send进行消息发送；ceph_con_send会唤醒工作任务执行消息发送；
>* (5) 接收消息时，内核网络协议栈在收到网卡消息后，会通过ceph注册的回调唤醒对应的connection worker进行收包，并调用初始化时绑定的消息处理回调函数；
>* (6) mon_client关闭会话时使用ceph_con_close关闭连接；

##### **2.1.4. 打开会话时客户端与monitor的交互(消息处理回调mon_con_ops分析)**

&emsp;&emsp;目前monitor对我们来说还是一个黑盒，因此无法分析其内部是如何处理客户端发送的hello消息的。那么我们如何才能获知客户端是如何与monitor进行交互的？这里我们可以打开内核动态日志开关(代码中有很多dout语句)，并通过分析日志来获知整个交互过程。通过打开libceph的messenger模块日志，我们可以得到如下图所示的消息交互过程：

<div align="center">
<img src="/images/posts/i440fx/rbd_5.jpg" height="550" width="400">  
</div> 

&emsp;&emsp;从上图可见，TCP连接建立后的前几次消息交互主要是messenger模块对连接进行初始化，我们在将在深入分析messenger模块时分析。ceph connection初始化完成后，客户端首先便是向monitor发送Auth(hello)认证消息，关于认证的实现细节我们这里暂不深入分析，通过日志可知，整个认证会有三次Auth消息的交互：第一次Auth_reply返回前，monitor会向客户端返回monmap；第三次Auth_reply返回后，表示认证成功。认证成功之后客户端会向monitor发送Subscribe订阅消息，表示关注osdmap的更新。由于是首次认证，monitor会向client返回初始的osdmap，后续只有在osdmap有更新时，monitor才回返回更新的map。

&emsp;&emsp;下面我们打开mon_con_ops看看客户端收到各类消息的处理过程：

```
linux/net/ceph/mon_client.c:

static const struct ceph_connection_operations mon_con_ops = {
    ...
    .dispatch = dispatch, /*收到monitor的消息时会调用该函数*/
    ...
};

/*
 * handle incoming message
 */
static void dispatch(struct ceph_connection *con, struct ceph_msg *msg)
{
    struct ceph_mon_client *monc = con->private;
    int type = le16_to_cpu(msg->hdr.type);

    if (!monc)
        return;

    /*这里针对不同消息进行不同处理*/
    switch (type) {
    case CEPH_MSG_AUTH_REPLY:
        handle_auth_reply(monc, msg);
        break;

    case CEPH_MSG_MON_SUBSCRIBE_ACK:
        handle_subscribe_ack(monc, msg);
        break;

    case CEPH_MSG_STATFS_REPLY:
        handle_statfs_reply(monc, msg);
        break;

    case CEPH_MSG_POOLOP_REPLY:
        handle_poolop_reply(monc, msg);
        break;

    case CEPH_MSG_MON_MAP:
        ceph_monc_handle_map(monc, msg);
        break;

    case CEPH_MSG_OSD_MAP:
        ceph_osdc_handle_map(&monc->client->osdc, msg);
        break;

    default:
        /* can the chained handler handle it? */
        if (monc->client->extra_mon_dispatch &&
                monc->client->extra_mon_dispatch(monc->client, msg) == 0)
            break;

        pr_err("received unknown message type %d %s\n", type,
        ceph_msg_type_name(type));
    }
    ceph_msg_put(msg);
}

/*处理monmap消息，接收最新的monitor信息*/
static void ceph_monc_handle_map(struct ceph_mon_client *monc, struct ceph_msg *msg)
{
    struct ceph_client *client = monc->client;
    struct ceph_monmap *monmap = NULL, *old = monc->monmap;
    void *p, *end;
    ...

    dout("handle_monmap\n");
    p = msg->front.iov_base;
    end = p + msg->front.iov_len;

    /*从收到的消息中解析出monmap内容，它包含所有monitor节点的IP信息*/
    monmap = ceph_monmap_decode(p, end);
    ...

    /*更新monmap*/
    client->monc.monmap = monmap;
    kfree(old);
    ...

out_unlocked:
    /*唤醒所有在auth_wq中等待的任务*/
    wake_up_all(&client->auth_wq);
}

/*处理osdmap消息，整体思路和monmap类似；这里分了增量和全量两种模式；
  osdmap中记录了所有osd的IP和状态信息*/
/*
* Process updated osd map.
*
* The message contains any number of incremental and full maps, normally
* indicating some sort of topology change in the cluster.  Kick requests
* off to different OSDs as needed.
*/
void ceph_osdc_handle_map(struct ceph_osd_client *osdc, struct ceph_msg *msg)
{
    ...
}

static void handle_auth_reply(struct ceph_mon_client *monc, struct ceph_msg *msg)
{
    int ret;
    int was_auth = 0;

    ...
    /*处理auth_reply消息*/
    ret = ceph_handle_auth_reply(monc->auth, msg->front.iov_base,
                msg->front.iov_len,
                monc->m_auth->front.iov_base,
                monc->m_auth->front_alloc_len);
    if (ret < 0) {
        /*如果认证出错，则唤醒等待任务并返回错误信息*/
        monc->client->auth_err = ret;
        wake_up_all(&monc->client->auth_wq);
    } else if (ret > 0) {
        /*继续发送认证请求，从日志分析共会发送三次*/
        __send_prepared_auth_request(monc, ret);
    } else if (!was_auth && ceph_auth_is_authenticated(monc->auth)) {
        /*认证成功，发送针对osdmap的订阅消息*/
        dout("authenticated, starting session\n");

        monc->client->msgr.inst.name.type = CEPH_ENTITY_TYPE_CLIENT;
        monc->client->msgr.inst.name.num =
        cpu_to_le64(monc->auth->global_id);

        __send_subscribe(monc);
        __resend_generic_request(monc);
    }
    ...

}
```

#### **2.2. rbd_dev_image_probe**

&emsp;&emsp;通过前文的分析我们可知，当monitor返回monmap和osdmap后，rbd进程将继续往下执行到rbd_dev_image_probe，这里将从存放对象的OSD中获取与当前RBD设备相关的信息：

```
linux/drivers/block/rbd.c:

/*rbd_dev已分配内存空间，mapping为true*/
static int rbd_dev_image_probe(struct rbd_device *rbd_dev, bool mapping)
{
    int ret;
    int tmp;

    /*从rados OSD池中获取当前RBD设备的image id，即对象"rbd_id.[RBD设备名称]"的内容*/
    ret = rbd_dev_image_id(rbd_dev);
    ...

    /*构造当前RBD设备元数据对象的名称，即"rbd_header.[image id]"*/
    ret = rbd_dev_header_name(rbd_dev);
    ...

    if (mapping) {
        ret = rbd_dev_header_watch_sync(rbd_dev, true);
        ...
    }

    if (rbd_dev->image_format == 1)
        ret = rbd_dev_v1_header_info(rbd_dev);
    else
        /*从rados OSD池中获取肖前RBD设备元数据对象的omap信息*/
        ret = rbd_dev_v2_header_info(rbd_dev);
    ...

    return 0;

    ...
}
```

##### **2.2.1. rbd_dev_image_probe -> rbd_dev_image_id**

```
linux/drivers/block/rbd.c:

static int rbd_dev_image_id(struct rbd_device *rbd_dev)
{
    int ret;
    size_t size;
    char *object_name;
    void *response;
    char *image_id;

    ...

    /*对象名称为RBD_ID_PREFIX("rbd_id.")+image_name*/
    size = sizeof (RBD_ID_PREFIX) + strlen(rbd_dev->spec->image_name);
    object_name = kmalloc(size, GFP_NOIO);
    ...

    /*返回的对象内容是一个经过编码的字符串，前四个字节代表后续内容的字节长度*/
    size = sizeof (__le32) + RBD_IMAGE_ID_LEN_MAX;
    response = kzalloc(size, GFP_NOIO);
    ...

    /*调用rbd同步对象访问接口，这里本质就是获取"rbd_id.RBD名称"对象的内容：
      (1)rbd_device为rbd_dev，即当前RBD设备；
      (2)接口访问对象名称为object_name，即"rbd_id.RBD名称"；
      (3)接口访问类(class)名称为"rbd"；
      (4)接口访问方法(method)名称为"get_id"；
      (5)接口访问输入参数为空；
      (6)接口访问输入参数长度为0；
      (7)接口访问输出结果保存内存位置为response；
      (8)接口访问输出结果保存内存最大长度为64。*/
    ret = rbd_obj_method_sync(rbd_dev, object_name,
                "rbd", "get_id", NULL, 0,
                response, RBD_IMAGE_ID_LEN_MAX);

    if (ret == -ENOENT) {
        ...
    } else if (ret > sizeof (__le32)) {
        void *p = response;

        /*从返回结果中解析出对象内容，即image_id*/
        image_id = ceph_extract_encoded_string(&p, p + ret, NULL, GFP_NOIO);
        ret = IS_ERR(image_id) ? PTR_ERR(image_id) : 0;
        if (!ret)
            rbd_dev->image_format = 2;
    } else {
        ret = -EINVAL;
    }

    if (!ret) {
        rbd_dev->spec->image_id = image_id;
        dout("image_id is %s\n", image_id);
    }
    ...
    return ret;
}
```

##### **2.2.2. rbd_dev_image_probe -> rbd_dev_header_name**

```
linux/drivers/block/rbd.c:

static int rbd_dev_header_name(struct rbd_device *rbd_dev)
{
    struct rbd_spec *spec = rbd_dev->spec;
    size_t size;

    ...
    /*header_name表示RBD设备对应头部对象(其omap保存了RBD设备元数据信息)名称
      即"rbd_header."+image_id*/
    size = sizeof (RBD_HEADER_PREFIX) + strlen(spec->image_id);

    rbd_dev->header_name = kmalloc(size, GFP_KERNEL);
    ...
    sprintf(rbd_dev->header_name, "%s%s", RBD_HEADER_PREFIX, spec->image_id);
    return 0;
}
```

##### **2.2.3. rbd_dev_image_probe -> rbd_dev_v2_header_info**

```
linux/drivers/block/rbd.c:

/*这里我们忽略与快照特性(snapshot)相关代码*/
static int rbd_dev_v2_header_info(struct rbd_device *rbd_dev)
{
    bool first_time = rbd_dev->header.object_prefix == NULL;
    int ret;

    down_write(&rbd_dev->header_rwsem);

    ret = rbd_dev_v2_image_size(rbd_dev);
    ...

    if (first_time) {
        ret = rbd_dev_v2_header_onetime(rbd_dev);
        ...
    }
    
    ...
    up_write(&rbd_dev->header_rwsem);

    return ret;
}

static int rbd_dev_v2_image_size(struct rbd_device *rbd_dev)
{
    return _rbd_dev_v2_snap_size(rbd_dev, CEPH_NOSNAP,
            &rbd_dev->header.obj_order,
            &rbd_dev->header.image_size);
}

static int _rbd_dev_v2_snap_size(struct rbd_device *rbd_dev, u64 snap_id,
        u8 *order, u64 *snap_size)
{
    __le64 snapid = cpu_to_le64(snap_id);
    int ret;
    struct {
        u8 order;
        __le64 size;
    } __attribute__ ((packed)) size_buf = { 0 };

    /*调用rbd同步对象访问接口，这里本质就是获取“rbd_header.image_id"对象中omap对应key的内容：
      (1)rbd_device为rbd_dev，即当前RBD设备；
      (2)接口访问对象名称为“rbd_header.image_id"；
      (3)接口访问类(class)名称为"rbd"；
      (4)接口访问方法(method)名称为"get_size"，对应omap中的key为"size"；
      (5)接口访问输入参数为snapid；
      (6)接口访问输入参数长度为snapid的字节长度；
      (7)接口访问输出结果保存内存位置为size_buf；
      (8)接口访问输出结果保存内存最大长度为size_buf空间大小。*/
    ret = rbd_obj_method_sync(rbd_dev, rbd_dev->header_name,
            "rbd", "get_size",
            &snapid, sizeof (snapid),
            &size_buf, sizeof (size_buf));
    dout("%s: rbd_obj_method_sync returned %d\n", __func__, ret);
    ...

    if (order) {
        *order = size_buf.order;
        dout("  order %u", (unsigned int)*order);
    }
    *snap_size = le64_to_cpu(size_buf.size);

    ...

    return 0;
}

static int rbd_dev_v2_header_onetime(struct rbd_device *rbd_dev)
{
    int ret;

    /*通过rbd_obj_method_sync获取头部对象omap中的"object_prefix"key的内容*/
    ret = rbd_dev_v2_object_prefix(rbd_dev);
    if (ret)
        goto out_err;

    /*通过rbd_obj_method_sync获取头部对象omap中的"features"key的内容*/
    ret = rbd_dev_v2_features(rbd_dev);
    if (ret)
        goto out_err;

    ...

    return ret;
}
```

##### **2.2.4. rbd_obj_method_sync**

&emsp;&emsp;通过前文分析，我们看到多个子函数中均使用了rbd_obj_method_sync函数来访问rados对象内容。该函数是对远程rados对象的同步访问接口，是对底层libceph接口(osd_client)的封装：

```
linux/drivers/block/rbd.c:

/*
 * Synchronous osd object method call.  Returns the number of bytes
 * returned in the outbound buffer, or a negative error code.
 */
/*osd对象的同步方法调用，返回结果在inbound中。注：上文的注释有问题*/
static int rbd_obj_method_sync(struct rbd_device *rbd_dev,  /*当前RBD设备*/
                        const char *object_name,            /*访问对象名称*/
                        const char *class_name,             /*访问类名称*/
                        const char *method_name,            /*访问方法名称*/
                        const void *outbound,               /*输入参数内存位置*/
                        size_t outbound_size,               /*输入参数大小*/
                        void *inbound,                      /*返回结果内存位置*/
                        size_t inbound_size)                /*返回结果大小*/
{
    struct ceph_osd_client *osdc = &rbd_dev->rbd_client->client->osdc; /*osd_client是底层libceph接口对象*/
    struct rbd_obj_request *obj_request;
    struct page **pages;
    u32 page_count;
    int ret;

    /*
     * Method calls are ultimately read operations.  The result
     * should placed into the inbound buffer provided.  They
     * also supply outbound data--parameters for the object
     * method.  Currently if this is present it will be a
     * snapshot id.
     */
    /*上面这段注释的意思是说针对对象的method call最终其实是对象的读操作。
      outbound表示method输入参数，当前只支持snapid；
      inbound表示返回结果。*/

    /*为返回结果inbound分配新的内存页， 为何要重新分配内存？*/
    page_count = (u32)calc_pages_for(0, inbound_size);
    pages = ceph_alloc_page_vector(page_count, GFP_KERNEL);
    if (IS_ERR(pages))
        return PTR_ERR(pages);

    ret = -ENOMEM;
    /*创建新的rbd_obj_request对象并初始化*/
    obj_request = rbd_obj_request_create(object_name, 0, inbound_size, OBJ_REQUEST_PAGES);
    if (!obj_request)
        goto out;

    obj_request->pages = pages;
    obj_request->page_count = page_count;

    /*创建rbd_obj_request在libceph对应的osd_request对象*/
    obj_request->osd_req = rbd_osd_req_create(rbd_dev, false, obj_request);
    if (!obj_request->osd_req)
        goto out;
    
    /*初始化osd_request中的op对象，它的类型为CEPH_OSD_OP_CALL，表示调用远端OSD中对象的方法*/
    osd_req_op_cls_init(obj_request->osd_req, 0, CEPH_OSD_OP_CALL, class_name, method_name);
    if (outbound_size) {
        /*如果存在输入参数outbound，则为其分配新的内存页并复制其内容*/

        struct ceph_pagelist *pagelist;

        pagelist = kmalloc(sizeof (*pagelist), GFP_NOFS);
        if (!pagelist)
            goto out;

        ceph_pagelist_init(pagelist);
        ceph_pagelist_append(pagelist, outbound, outbound_size); /*将outbound中内容复制到pagelist*/
        osd_req_op_cls_request_data_pagelist(obj_request->osd_req, 0,pagelist); /*将pagelist作为op的输入参数(request_data)*/
    }
    osd_req_op_cls_response_data_pages(obj_request->osd_req, 0,
            obj_request->pages, inbound_size, 0, false, false);/*将obj_request->pages作为op的输出结果(response_data)*/
    rbd_osd_req_format_read(obj_request);

    /*将rbd_object_request对象提交给底层libceph发送给远端osd节点*/
    ret = rbd_obj_request_submit(osdc, obj_request);
    if (ret)
        goto out;

    /*等待远端osd节点返回响应*/
    ret = rbd_obj_request_wait(obj_request);
    if (ret)
        goto out;

    /*远端osd节点返回响应后，result中保存返回结果*/
    ret = obj_request->result;
    if (ret < 0)
        goto out;

    rbd_assert(obj_request->xferred < (u64)INT_MAX);
    ret = (int)obj_request->xferred;
    /*根据远端osd返回的数据量xferred，将数据拷贝到inbound中*/
    ceph_copy_from_page_vector(pages, inbound, 0, obj_request->xferred);
out:
    if (obj_request)
        rbd_obj_request_put(obj_request);
    else
        ceph_release_page_vector(pages, page_count);

return ret;
}
```

&emsp;&emsp;rbd_obj_request_submit中涉及ceph的数据分布CRUSH算法，我们将在介绍IO流程中对此展开分析。

#### **2.3. rbd_dev_device_setup**

&emsp;&emsp;基于之前获取的信息，我们可以通过内核块层提供的相关接口创建一个新的RBD块设备对象供应用使用：

```
linux/drivers/block/rbd.c:

static int rbd_dev_device_setup(struct rbd_device *rbd_dev)
{
    int ret;

    /*首先根据全局变量rbd_dev_id_max生成一个新的最大id号*/
    /* generate unique id: find highest unique id, add one */
    rbd_dev_id_get(rbd_dev);

    /*根据id号生成设备名称*/
    /* Fill in the device name, now that we have its id. */
    BUILD_BUG_ON(DEV_NAME_LEN < sizeof (RBD_DRV_NAME) + MAX_INT_FORMAT_WIDTH);
    sprintf(rbd_dev->name, "%s%d", RBD_DRV_NAME, rbd_dev->dev_id);

    /* Get our block major device number. */

    /*通过块层接口register_blkdev注册一个新的块设备*/
    ret = register_blkdev(0, rbd_dev->name);
    if (ret < 0)
        goto err_out_id;
    rbd_dev->major = ret;

    /* Set up the blkdev mapping. */

    /*调用块层接口初始化gendisk对象，绑定其IO处理函数为rbd_request_fn(我们将以此为入口来分析IO流程)*/
    ret = rbd_init_disk(rbd_dev);
    if (ret)
        goto err_out_blkdev;

    /*快照相关，暂不关心*/
    ret = rbd_dev_mapping_set(rbd_dev);
    if (ret)
        goto err_out_disk;
    /*设备容量*/
    set_capacity(rbd_dev->disk, rbd_dev->mapping.size / SECTOR_SIZE);

    /*在sys目录下添加设备*/
    ret = rbd_bus_add_dev(rbd_dev);
    if (ret)
        goto err_out_mapping;

    /* Everything's ready.  Announce the disk to the world. */

    /*通过add_disk接口在系统中呈现一个新的块设备*/
    set_bit(RBD_DEV_FLAG_EXISTS, &rbd_dev->flags);
    add_disk(rbd_dev->disk);

    pr_info("%s: added with size 0x%llx\n", rbd_dev->disk->disk_name,
        (unsigned long long) rbd_dev->mapping.size);

    return ret;

err_out_mapping:
    rbd_dev_mapping_clear(rbd_dev);
err_out_disk:
    rbd_free_disk(rbd_dev);
err_out_blkdev:
    unregister_blkdev(rbd_dev->major, rbd_dev->name);
err_out_id:
    rbd_dev_id_put(rbd_dev);
    rbd_dev_mapping_clear(rbd_dev);

    return ret;
}
```

### 3. RBD块设备IO流程分析

&emsp;&emsp;我们针对IO流程同样沿着从上至下的顺序进行深入分析，对于和上节重复部分我们将不再展开，重点讨论有差异的部分。 IO流程可分为请求下发和响应返回两个阶段。

#### **3.1. IO下发：rbd_request_fn**

```
linux/drivers/block/rbd.c:

/*当应用向RBD块设备下发读写请求时，最终会调用该函数；
  q代表请求所在的请求队列*/
static void rbd_request_fn(struct request_queue *q)
__releases(q->queue_lock) __acquires(q->queue_lock)
{
    struct rbd_device *rbd_dev = q->queuedata; /*队列私有数据，初始化时指定，代表rbd_dev对象*/
    struct request *rq;
    int result;

    /*通过一个大循环，不断调用blk_fetch_request从请求队列中取出待处理的请求rq*/
    while ((rq = blk_fetch_request(q))) {
        bool write_request = rq_data_dir(rq) == WRITE; /*判断请求读写类型*/
        struct rbd_img_request *img_request;
        u64 offset;
        u64 length;

        ...
        offset = (u64) blk_rq_pos(rq) << SECTOR_SHIFT; /*计算请求起始位置，从扇区转换成字节*/
        length = (u64) blk_rq_bytes(rq); /*获取请求大小*/
        ...

        result = -ENOMEM;
        /*针对每个rq，这里会生成一个img_request与之对应；其内部会记录请求起始位置、长度和读写类型等信息*/
        img_request = rbd_img_request_create(rbd_dev, offset, length, write_request);
        if (!img_request)
            goto end_request;
        
        /*绑定img_request和rq之间的关系*/
        img_request->rq = rq;

        /*向img_request中填入bio信息，下面将展开分析*/
        result = rbd_img_request_fill(img_request, OBJ_REQUEST_BIO, rq->bio);
        /*将img_request递交给底层libceph过行网络发送，下面将展开分析*/
        if (!result)
            result = rbd_img_request_submit(img_request);
        ...
    }
}
```

##### **3.1.1. rbd_img_request_fill**

&emsp;&emsp;由于每个RBD设备会以4M为粒度划分成一个个对象，所以对每个RBD设备的请求(image request)会转换成若干对象请求(object request)。rbd_img_request_fill的主要作用就是完成image request到object request的转换：

```
linux/drivers/block/rbd.c:

static int rbd_img_request_fill(struct rbd_img_request *img_request,
                        enum obj_request_type type,void *data_desc)
{
    struct rbd_device *rbd_dev = img_request->rbd_dev;
    struct rbd_obj_request *obj_request = NULL;
    struct rbd_obj_request *next_obj_request;
    bool write_request = img_request_write_test(img_request); /*是否为写请求*/
    struct bio *bio_list = 0;
    unsigned int bio_offset = 0;
    struct page **pages = 0;
    u64 img_offset;
    u64 resid;
    u16 opcode;

    
    opcode = write_request ? CEPH_OSD_OP_WRITE : CEPH_OSD_OP_READ;
    img_offset = img_request->offset; /*image request对应的请求偏移*/
    resid = img_request->length; /*image request对应的请求长度*/
    rbd_assert(resid > 0);

    if (type == OBJ_REQUEST_BIO) {/*这里主要考虑bio类型的请求*/
        bio_list = data_desc;
        rbd_assert(img_offset == bio_list->bi_sector << SECTOR_SHIFT);
    } else {
        ...
    }

    while (resid) {
        struct ceph_osd_request *osd_req;
        const char *object_name;
        u64 offset;
        u64 length;

        /*根据image request偏移位置计算对象名称*/
        object_name = rbd_segment_name(rbd_dev, img_offset);
        ...
        /*4M对象内的偏移*/
        offset = rbd_segment_offset(rbd_dev, img_offset);
        /*4M对象内的长度*/
        length = rbd_segment_length(rbd_dev, img_offset, resid);
        /*新建一个object request*/
        obj_request = rbd_obj_request_create(object_name, offset, length, type);
        /* object request has its own copy of the object name */
        rbd_segment_name_free(object_name);
        if (!obj_request)
            goto out_unwind;
        /*
         * set obj_request->img_request before creating the
         * osd_request so that it gets the right snapc
         */
        rbd_img_obj_request_add(img_request, obj_request);

        if (type == OBJ_REQUEST_BIO) {
            unsigned int clone_size;

            rbd_assert(length <= (u64)UINT_MAX);
            clone_size = (unsigned int)length;
            /*克隆object request对应的bio段*/
            obj_request->bio_list =
                bio_chain_clone_range(&bio_list,
                                    &bio_offset,
                                    clone_size,
                                    GFP_ATOMIC);
            ...
        } else {
            ...
        }

        /*针对object request创建底层的osd request*/
        osd_req = rbd_osd_req_create(rbd_dev, write_request, obj_request);
        ...
        obj_request->osd_req = osd_req;
        obj_request->callback = rbd_img_obj_callback; /*请求处理完成时的回调*/
        rbd_img_request_get(img_request);

        /*将object request的bio信息填入到osd request的extent中*/
        osd_req_op_extent_init(osd_req, 0, opcode, offset, length, 0, 0);
        if (type == OBJ_REQUEST_BIO)
            osd_req_op_extent_osd_data_bio(osd_req, 0, obj_request->bio_list, length);
        else
            ...

        if (write_request)
            rbd_osd_req_format_write(obj_request);
        else
            rbd_osd_req_format_read(obj_request);

        obj_request->img_offset = img_offset;

        /*更新偏移和长度，并继续循环*/
        img_offset += length; 
        resid -= length;
    }

    return 0;

    ...
}
```

##### **3.1.2. rbd_img_request_submit**

&emsp;&emsp;完成object request对象的成生后，rbd_image_request_submit调用rbd_obj_request_submit，最终来到libceph中的ceph_osdc_start_request，它的主要作用就是根据访问对象、OSDMap和CRUSH算法计算出对象所在PG对应的主OSD节点，并将该object request对应的osd request通过网络发送给该OSD处理：

```
linux/net/ceph/osd_client.c:

int ceph_osdc_start_request(struct ceph_osd_client *osdc,
            struct ceph_osd_request *req, bool nofail)
{
    int rc = 0;

    down_read(&osdc->map_sem);
    mutex_lock(&osdc->request_mutex);
    /*将该osd请求添加到osdc中*/
    __register_request(osdc, req);
    req->r_sent = 0;
    req->r_got_reply = 0;
    /*通过CRUSH算法及OSDMap映射当前osd请求到OSD节点，有关CRUSH的原理可以参考相关论文*/
    rc = __map_request(osdc, req, 0);
    if (rc < 0) {
        ...
    }
    if (req->r_osd == NULL) {
        dout("send_request %p no up osds in pg\n", req);
        ceph_monc_request_next_osdmap(&osdc->client->monc);
    } else {
        /*将该请求发送给OSD节点*/
        __send_queued(osdc);
    }
    rc = 0;
out_unlock:
    mutex_unlock(&osdc->request_mutex);
    up_read(&osdc->map_sem);
    return rc;
}
```

#### **3.2. IO返回**

&emsp;&emsp;IO请求通过网络发送给OSD节点后，OSD节点在处理完成后会回复响应，响应的处理逻辑在客户端创建OSD对象时指定：

```
linux/net/ceph/osd_client.c:

static struct ceph_osd *create_osd(struct ceph_osd_client *osdc, int onum)
{
    struct ceph_osd *osd;

    osd = kzalloc(sizeof(*osd), GFP_NOFS);
    if (!osd)
        return NULL;

    atomic_set(&osd->o_ref, 1);
    osd->o_osdc = osdc;
    osd->o_osd = onum;
    RB_CLEAR_NODE(&osd->o_node);
    INIT_LIST_HEAD(&osd->o_requests);
    INIT_LIST_HEAD(&osd->o_linger_requests);
    INIT_LIST_HEAD(&osd->o_osd_lru);
    osd->o_incarnation = 1;

    ceph_con_init(&osd->o_con, osd, &osd_con_ops, &osdc->client->msgr);

    INIT_LIST_HEAD(&osd->o_keepalive_item);
    return osd;
}

static const struct ceph_connection_operations osd_con_ops = {
    .get = get_osd_con,
    .put = put_osd_con,
    .dispatch = dispatch, /*消息处理逻辑*/
    .get_authorizer = get_authorizer,
    .verify_authorizer_reply = verify_authorizer_reply,
    .invalidate_authorizer = invalidate_authorizer,
    .alloc_msg = alloc_msg,
    .fault = osd_reset,
};

static void dispatch(struct ceph_connection *con, struct ceph_msg *msg)
{
    struct ceph_osd *osd = con->private;
    struct ceph_osd_client *osdc;
    int type = le16_to_cpu(msg->hdr.type);

    if (!osd)
        goto out;
    osdc = osd->o_osdc;

    switch (type) {
    case CEPH_MSG_OSD_MAP:
        ceph_osdc_handle_map(osdc, msg);
        break;
    case CEPH_MSG_OSD_OPREPLY:
        handle_reply(osdc, msg, con);
        break;
    case CEPH_MSG_WATCH_NOTIFY:
        handle_watch_notify(osdc, msg);
        break;

    default:
        pr_err("received unknown message type %d %s\n", type,
        ceph_msg_type_name(type));
    }
out:
    ceph_msg_put(msg);
}
```

&emsp;&emsp;在handle_reply中会从osd request到object request，再到image request，层层调用回调并处理已经结束的IO请求。

### 4. libceph.ko中messenger模块分析

&emsp;&emsp;messenger模块(信使)是libceph中相对比较独立的部分，旨在为上层各种网络客户端(如mon_client、osd_client)提供稳定可靠、有序的网络服务。messenger构建在网络TCP协议之上，虽然TCP协议本身是面向连接且可靠的网络协议，但是TCP连接有可能断开(broken，非长连接情况下会发生；长连接存在底层网络故障排除后上层应用感知不及时的问题)。为解决TCP短连接不可靠的问题，messenger通过间隔性地重连方案(back off，当有消息要发送时)或者等待方案(stand by，当无消息要发送时)来解决。同时会给每个发送的消息带上唯一的序列号(seq)，用以区分是否为重复消息。

&emsp;&emsp;messenger模块内部架构如下图所示：

<div align="center">
<img src="/images/posts/i440fx/rbd_6.jpg" height="400" width="700">  
</div> 

>* 从发送流程来看(图中实线)，应用程序(如rbd命令)调用ceph_con_send进行消息发送；该函数会唤醒发送连接对应的工作任务；在工作任务进程中，通过try_write函数调用kernel_sendmsg访问底层网络协议栈，最终触发网卡发包；
>* 从接收流程来看(图中虚线)，网卡收包后通过中断和协议栈回调唤醒工作任务；工作任务通过try_read调用kernel_recvmsg从网络协议栈中读取消息内容；最后调用连接初始化时指定的消息回调函数对消息进行处理。

#### **4.1. 正常网络收发的处理**

##### **4.1.1 socket连接阶段**

&emsp;&emsp;每当上层通过ceph_con_init创建一个ceph_connection对象(对底层TCP连接的封装，含有自身的状态变化)并调用ceph_con_open打开该连接后(连接状态变为PREOPEN)，就会在内核工作队列中添加一项新的任务，并对该任务进行一次调度，其入口函数为con_work，代码框架如下所示：

```
linux/net/ceph/messenger.c:

static void con_work(struct work_struct *work)
{
    struct ceph_connection *con = container_of(work, struct ceph_connection,
        work.work);

    ...
    while (true) {
        ...
        ret = try_read(con); /*socket连接初始化阶段不进行任何工作*/
        ...
        ret = try_write(con); /*建立socket连接并设定socket回调函数*/
        ...
        break;
    }
    ...
}
```

&emsp;&emsp;conection worker首次被调度时进入socket连接阶段，完成的主要工作是建立TCP socket连接，并将ceph_connection的状态置为CONNECTING：

```
linux/net/ceph/messenger.c:

static int try_write(struct ceph_connection *con)
{
    ...

    if (con->state == CON_STATE_PREOPEN) { /*首次被调度时，连接状态为PREOPEN*/
        BUG_ON(con->sock);
        con->state = CON_STATE_CONNECTING; /*执行完下面这些动作后状态更新为CONNECTING*/

        con_out_kvec_reset(con); /*重置当前连接con的发送缓冲区，对应con->out_kvec_* */
        prepare_write_banner(con); /*将客户端的banner内容放入发送缓冲区*/
        prepare_read_banner(con); /*准备接收服务端的banner内容*/

        BUG_ON(con->in_msg);
        con->in_tag = CEPH_MSGR_TAG_READY; /*重置消息接收状态为READY，表示可接收任意消息类别，不同类别由CEPH_MSGR_TAG_*进行区分 */
        ret = ceph_tcp_connect(con); /*创建并连接socket，同时指定协议栈的回调函数*/
        if (ret < 0) {
            con->error_msg = "connect error";
            goto out;
        }
    }

more_kvec:
    ...
    if (con->out_kvec_left) { 
        /*调用底层kernel_sendmsg将当前连接con发送缓冲中的内容进行发送*/
        ret = write_partial_kvec(con);
        if (ret <= 0)
            goto out;
    }
    ...
}

static int ceph_tcp_connect(struct ceph_connection *con)
{
    struct sockaddr_storage *paddr = &con->peer_addr.in_addr;
    struct socket *sock;
    int ret;

    BUG_ON(con->sock);
    ret = sock_create_kern(con->peer_addr.in_addr.ss_family, SOCK_STREAM,
        IPPROTO_TCP, &sock);
    ...
    sock->sk->sk_allocation = GFP_NOFS;

    set_sock_callbacks(sock, con);

    con_sock_state_connecting(con);
    ret = sock->ops->connect(sock, (struct sockaddr *)paddr, sizeof(*paddr),
        O_NONBLOCK);
    ...
    con->sock = sock;
    return 0;
}

static void set_sock_callbacks(struct socket *sock,
    struct ceph_connection *con)
{
    struct sock *sk = sock->sk;
    sk->sk_user_data = con;
    sk->sk_data_ready = ceph_sock_data_ready; /*协议栈收到包后会回调该函数*/
    sk->sk_write_space = ceph_sock_write_space;
    sk->sk_state_change = ceph_sock_state_change; /*协议栈改变socket状态时会回调该函数*/
}

static void ceph_sock_data_ready(struct sock *sk, int count_unused)
{
    struct ceph_connection *con = sk->sk_user_data;

    ...
    queue_con(con); /*再次唤醒工作任务*/
    ...
}
```

##### **4.1.2 协商阶段**

&emsp;&emsp;socket连接阶段最后，向服务端发送了客户端的banner，并等待服务端回复banner。当收到服务端回复banner后，connection worker再次被唤醒，此时主要完成客户端与服务端的信息协商：

```
linux/net/ceph/messenger.c:

static void con_work(struct work_struct *work)
{
    struct ceph_connection *con = container_of(work, struct ceph_connection,
        work.work);

    ...
    while (true) {
        ...
        ret = try_read(con); /*读取服务端返回的banner并进入协商阶段*/
        ...
        ret = try_write(con); /*向服务端发送协商信息*/
        ...
        break;
    }
    ...
}

static int try_read(struct ceph_connection *con)
{
    ...
    if (con->state == CON_STATE_CONNECTING) {
        ret = read_partial_banner(con); /*读取服务端的banner信息*/
        if (ret <= 0)
            goto out;
        ret = process_banner(con); /*对banner进行校验*/
        if (ret < 0)
            goto out;

        con->state = CON_STATE_NEGOTIATING; /*校验通过后进入协商阶段*/

        /*
         * Received banner is good, exchange connection info.
         * Do not reset out_kvec, as sending our banner raced
         * with receiving peer banner after connect completed.
         */
        ret = prepare_write_connect(con); /*准备全局连接号等协商信息*/
        if (ret < 0)
            goto out;
        prepare_read_connect(con); /*准备接收服务端返回的协商信息*/

        /* Send connection info before awaiting response */
        goto out;
    }
    ...
｝
```

##### **4.1.3 正常打开阶段**

&emsp;&emsp;收到服务端的协商消息后，connection worker再次被唤醒，进行服务端协商消息的处理并进入正常打开阶段可收发消息：

```
linux/net/ceph/messenger.c:

static void con_work(struct work_struct *work)
{
    struct ceph_connection *con = container_of(work, struct ceph_connection,
        work.work);

    ...
    while (true) {
        ...
        ret = try_read(con); /*读取服务端协商信息并进入正常打开阶段，可正常接收消息*/
        ...
        ret = try_write(con); /*可正常发送消息*/
        ...
        break;
    }
    ...
}

static int try_read(struct ceph_connection *con)
{
    ...
    if (con->state == CON_STATE_NEGOTIATING) {
        ret = read_partial_connect(con); /*读取服务端协商信息到in_reply中*/
        if (ret <= 0)
            goto out;
        ret = process_connect(con); /*处理协商消息并进入正常打开阶段*/
        if (ret < 0)
            goto out;
        goto more;
    }
    ...
}

static int process_connect(struct ceph_connection *con)
{
    ...
    switch (con->in_reply.tag) { /*in_reply中记录服务端返回的协商信息*/
        ...
    case CEPH_MSGR_TAG_SEQ:
    case CEPH_MSGR_TAG_READY:
        ...

        /*记录服务端返回的协商消息并将连接状态置为OPEN*/
    
        con->state = CON_STATE_OPEN;
        con->auth_retry = 0;    /* we authenticated; clear flag */
        con->peer_global_seq = le32_to_cpu(con->in_reply.global_seq);
        con->connect_seq++;
        con->peer_features = server_feat;
        ...
        con->delay = 0;      /* reset backoff memory */

        if (con->in_reply.tag == CEPH_MSGR_TAG_SEQ) {
            prepare_write_seq(con);
            prepare_read_seq(con);
        } else {
            prepare_read_tag(con);
        }
        break;
    }
}
```

##### **4.1.4 消息收发阶段**

&emsp;&emsp;客户端与服务端进行正常消息收发时，总是先在网络连接上发送一个字节的tag，再发送实际的消息。原因是两者可以通过tag来明确消息的具体类别和解析格式。我们先来看看消息的接收与处理：

```
linux/net/ceph/messenger.c:

static int try_read(struct ceph_connection *con)
{
    ...

    if (con->in_tag == CEPH_MSGR_TAG_READY) { /*READY代表准备好接收具体的tag内容*/
        /*
         * what's next?
         */
        ret = ceph_tcp_recvmsg(con->sock, &con->in_tag, 1); /*从网络中接收一个字节的tag*/
        if (ret <= 0)
            goto out;

        /*根据tag内容准备接收后续的消息*/
        switch (con->in_tag) {
        case CEPH_MSGR_TAG_MSG:
            prepare_read_message(con);
            break;
        case CEPH_MSGR_TAG_ACK:
            prepare_read_ack(con);
            break;
        case CEPH_MSGR_TAG_CLOSE:
            con_close_socket(con);
            con->state = CON_STATE_CLOSED;
            goto out;
        default:
            goto bad_tag;
        }
    }

    /*如果tag为MSG，则表示后续为一个实际的消息，开始接收消息内容并调用回调函数*/
    if (con->in_tag == CEPH_MSGR_TAG_MSG) {
        ret = read_partial_message(con);
        ...
        if (con->in_tag == CEPH_MSGR_TAG_READY)
            goto more;
        process_message(con);
        if (con->state == CON_STATE_OPEN)
            prepare_read_tag(con);
        goto more;
    }

    /*如果tag为ACK或SEQ，则表示后续为一个服务端返回的确认号，可以释放已确认的发送消息*/
    if (con->in_tag == CEPH_MSGR_TAG_ACK ||
        con->in_tag == CEPH_MSGR_TAG_SEQ) {
        /*
         * the final handshake seq exchange is semantically
         * equivalent to an ACK
         */
        ret = read_partial_ack(con);
        ...
        process_ack(con);
        goto more;
    }
    ...
}
```

&emsp;&emsp;接下来再来看看消息的发送：

```
linux/net/ceph/messenger.c:


static int try_write(struct ceph_connection *con)
{
more:
    ...
more_kvec:    
    if (con->out_kvec_left) { /*将发送缓冲区中的内容通过网络进行发送*/
        ret = write_partial_kvec(con);
        if (ret <= 0)
            goto out;
    }

    /* msg pages? */
    if (con->out_msg) { /*如果已选定当前发送消息out_msg且还未完成发送，则发送消息包含的数据*/
        if (con->out_msg_done) {
            ceph_msg_put(con->out_msg);
            con->out_msg = NULL;   /* we're done with this one */
            goto do_next;
        }

        ret = write_partial_message_data(con);
        if (ret == 1)
            goto more_kvec;  /* we need to send the footer, too! */
        if (ret == 0)
            goto out;
        if (ret < 0) {
            dout("try_write write_partial_message_data err %d\n", ret);
            goto out;
        }
    }

do_next:
    if (con->state == CON_STATE_OPEN) {
        /* is anything else pending? */
        if (!list_empty(&con->out_queue)) {
            /*如果发送队列out_queue中有待发送的消息，则取出一个消息放到out_msg中并发送其头部内容(消息中包含的数据在前段代码中发送)*/
            prepare_write_message(con);
            goto more;
        }
        if (con->in_seq > con->in_seq_acked) {
            /*如果当前收到消息的序列号大于已经确认的序列号，则准备发送新的确认号*/
            prepare_write_ack(con);
            goto more;
        }
        ...
    }
    ...
}
```
#### **4.2. TCP连接故障后的处理**

<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [Rados Block Device之二－客户端内核RBD驱动分析](https://rootw.github.io/2018/01/RBD-client/) 
