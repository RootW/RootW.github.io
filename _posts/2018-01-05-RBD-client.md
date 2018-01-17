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

&emsp;&emsp;从文件内容看，rbd_id.wb中保存的主要是个id信息，为"371e643c9869"。知道了wb块设备的id，我们再看来看看rbd_head.371e643c9869对象中的内容：

> [root@ceph-client]# rados get rbd_header.371e643c9869 ./result.txt --pool wbpool; hexdump -Cv ./result.txt  
>  

&emsp;&emsp;rbd_header.371e643c9869对象内容为空，怎么回事呢？我们再来看看该对象相关的key/value map值(可以用来描述对象的元数据信息)：

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

&emsp;&emsp;这样我们看到rbd_header.371e643c9869对象的map中保存与RBD相关的元数据信息，如RBD数据存放对象前缀为"rbd_data.371e643c9869"；order为0x16，表示以4M单位来划分RBD块设备，每4M对应一个对象，对象名为"rbd_data.371e643c9869.偏移"；size为0x40000000，即1G。

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

&emsp;&emsp;下面我们深入rbd_add函数，看看内核RBD驱动是如何进行块设备的生成的：

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

    /*从ceph集群返回的信息中获取当前pool对应的id*/
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

err_out_rbd_dev:
    rbd_dev_destroy(rbd_dev);
err_out_client:
    rbd_put_client(rbdc);
err_out_args:
    rbd_spec_put(spec);
err_out_module:
    module_put(THIS_MODULE);

    dout("Error adding device %s\n", buf);

    return (ssize_t)rc;
}

```

### 3. RBD块设备IO流程分析

CRUSH



<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [Rados Block Device之二－客户端内核RBD驱动分析](https://rootw.github.io/2018/01/RBD-client/) 
