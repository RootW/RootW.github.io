---
layout: post
title: 【firecracker】概述
date: 2019-08-15
tags: firecracker
---

### Ｗhat

&emsp;&emsp;firecracker（中文名鞭炮，寓意虚拟机的生命可以像鞭炮一样短暂，一闪即逝，但相同时刻却可以有成千上万个虚拟机快速闪现）是一种基于KVM的轻量虚拟化技术，相比普通虚拟机，firecracker虚拟机内存占用更少、启动速度更快；相比容器，firecracker则具有极强的安全隔离性。另外，firecracker采用全新的安全编程语言Rust实现，提供了语言级的内存安全与线程安全保障，使它天生具备了安全基因。

### Why

&emsp;&emsp;firecracker为了同时实现虚拟机的隔离性和容器的轻量性，也付出了一定的代价：由于裁剪了标准主板架构中的BIOS模块、PCI系统和ACPI模块，因此它不支持CPU和设备的热插拔；它是短生命周期的，就如同鞭炮，一闪即逝，因此不支持热迁移等长周期特性；虚拟机内部操作系统内核定制，不支持通用操作系统。所以firecracker目前瞄准的应用场景主要是作为serverless平台的运行时底座，为serverless函数提供轻量、安全的执行环境。

### How to Use

&emsp;&emsp;以X86平台为例，我们可以通过如下命令启动一个firecracker虚拟机：

&emsp;&emsp;1、启动firecracker进程
> firecracker --api-sock /tmp/firecracker.socket

&emsp;&emsp;2、配置vmlinux
> curl --unix-socket /tmp/firecracker.socket -i -X PUT 'http://localhost/boot-source' -H 'Accept: application/json' -H 'Content-Type: application/json' -d '{"kernel_image_path": "/root/hello-vmlinux.bin", "boot_args": "console=ttyS0 reboot=k panic=1 pci=off"}'

&emsp;&emsp;3、配置rootfs
> curl --unix-socket /tmp/firecracker.socket -i -X PUT 'http://localhost/drives/rootfs' -H 'Accept: application/json' -H 'Content-Type: application/json' -d '{"drive_id": "rootfs", "path_on_host": "/root/hello-rootfs.ext4", "is_root_device": true, "is_read_only": false}'

&emsp;&emsp;4、配置虚拟机并运行
> curl --unix-socket /tmp/firecracker.socket -i -X PUT 'http://localhost/machine-config' -H 'Accept: application/json' -H 'Content-Type: application/json' -d '{"vcpu_count": 2, "mem_size_mib": 1024, "ht_enabled": false}'
curl --unix-socket /tmp/firecracker.socket -i -X PUT 'http://localhost/actions' -H 'Accept: application/json' -H 'Content-Type: application/json' -d '{"action_type": "InstanceStart"}'

### How to Implement

#### **系统架构**

&emsp;&emsp;从整体架构上看，firecracker与普通KVM虚拟化非常相似：物理主机上直接运行Host OS系统，其内核中部署KVM模块；KVM内核模块为用户态虚拟化程序(firecracker)提供硬件辅助的核心部件(CPU、MMU、本地中断控制器与时钟系统等)虚拟化能力；firecracker模拟的机器只实现了运行应用所必需的组件，其内部运行的Guest OS系统是一个经过裁剪优化的运行时环境，可以支持绝大多数客户应用程序的执行。

<div align="center">                                                             
    <img src="/images/posts/firecracker/architecture.jpg" height="552" width="867">  
</div>

#### **设备模型与线程模型**

&emsp;&emsp;正如上节架构分析中所述，firecracker虚拟机仅包含了运行客户应用所必需的组件，是一种非常精简的设备模型，如下图所示仅包含 CPU、内存、时钟系统、中断系统、IO设备(virtio-blk/virtio-net)与输入输出串口。CPU与内存是核心运算和控制系统，客户应用对外呈现的绝大多数功能由它们提供；时钟系统为系统提供计时功能和定时通知功能，其中计时功能为内核和客户应用提供时间概念，定时通知功能可实现定时器功能并可作为内核调度机制的触发条件；中断系统为各类外设提供了一种向CPU报告事件发生的通用机制；IO设备为系统提供了基础存储或网络服务，使系统具备了“记忆”和“通信”能力；串口设备为虚拟机和系统管理员之间提供一种交互手段，使得虚拟机可以响应管理员下发的各种管理命令。

<div align="center">                                                             
    <img src="/images/posts/firecracker/machine_model.jpg" height="342" width="549">  
</div>

&emsp;&emsp;从线程模型上分析，firecracker和KVM上的qemu程序几乎一样(下图显示了一个2核firecracker虚拟机运行时生成的4个线程)：由虚拟CPU线程（图中fc_vcpu0和fc_vcpu1）实现虚拟机CPU和内存功能的模拟；由IO线程(图中fc_vmm)实现对各类IO设备的模拟(即响应虚拟机CPU发出的IO请求)；不同的是，firecracker有一个独立的API_Server线程(图中firecracker)，它基于http协议接收用户的命令并将命令传递给IO线程处理。

<div align="center">                                                             
    <img src="/images/posts/firecracker/thread_model.jpg" height="88" width="576">  
</div>

&emsp;&emsp;下面我们将以X86虚拟机为例通过一系列博文逐个分析firecracker各个子系统运行原理与初始化流程。

>* CPU与内存
>* 时钟与中断系统
>* Virtio-mmio设备
>* legacy设备(串口/键盘)
>* 初始化框架

<br
转载请注明：[吴斌的博客](https://rootw.github.io) » [【firecracker】概述](https://rootw.github.io/2019/08/firecracker-all/) 
