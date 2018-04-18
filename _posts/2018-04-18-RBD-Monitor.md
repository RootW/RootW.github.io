---
layout: post
title: 【Rados Block Device】九、Monitor原理分析
date: 2018-04-18
tags: ceph
---

&emsp;&emsp;Monitors是ceph集群的管理节点，负责维护整个集群的全局信息(如OSDMap)；Client和OSD加入和退出集群时，都需要和Monitors打交道，而且都需要从Monitors中获取最新的全局信息(如OSDMap)进行相关操作(如CRUSH数据映射)。 CatKang的[博文](https://www.jianshu.com/p/60b34ba5cdf2)对Monitor进行了比较多全的分析，这里我只补充一些自己的理解。Monitor的代码分析大家可以对照原理分析自行开展，略显枯燥，paxos算法相关原理可参考[此篇博文](https://www.cnblogs.com/linbingdong/p/6253479.html)，不过注意一点，ceph没有完全按照paxos来实现，作了一定的修改。

### 架构

&emsp;&emsp;Monitor整体架构如下所示，注意，这里没有体现网络层，其原理和OSD中分析的类似，请参考messenger模块分析：

<div align="center">
<img src="/images/posts/ceph/rbd_monitor_arch.jpg" height="280" width="400">  
</div> 

&emsp;&emsp;从总体处理流程上来看，Monitor首先也是通过messenger模块接收网络消息；接着对于不同的全局信息提供不同的PaxosService；但是对于这些服务都会提交给Paxos模块处理，该模块实现了核心的Paxos算法；对于更新请求，最终Paxos会将更新内容提交到底层的数据库中进行存储。

### 初始化流程

&emsp;&emsp;Monitor初始化整体流程如下图所示：

<div align="center">
<img src="/images/posts/ceph/rbd_monitor_init.jpg" height="450" width="600">  
</div> 

&emsp;&emsp;初始化过程中，网络messenger模块执行的动作和OSD是一样的，只不过这里只需要一个public messenger对象(Monitor不接入cluster网络平面)。消息的处理是由Monitor对象(类似OSD进程中的OSD对象)进行的，入口函数在Monitor::dispatch_op。

#### **1. bootstrap发现**

&emsp;&emsp;网络模块初始化完成后，Monitor首先进行的是bootstrap动作，通过网络协商的方式加入到Monitor集群中(quorum)：

<div align="center">
<img src="/images/posts/ceph/rbd_monitor_probe.jpg" height="400" width="600">  
</div> 

#### **2. election选主**

&emsp;&emsp;bootstrap发现其它Monitor节点后，将进行一轮选主动作，从所有Monitor中选出一个Leader，而其它Monitor就成为Peon(劳工)。选主的目的是只有Leader可以向所有Monitor发起信息变更请求，解决Paxos算法中的**活性**问题。所有Monitor都可以响应查询请求。选主流程如下：

<div align="center">
<img src="/images/posts/ceph/rbd_monitor_election.jpg" height="400" width="600">  
</div> 

#### **3. recovery恢复**

&emsp;&emsp;选主完成后，Leader将发起恢复动作，在所有Monitor之间进行数据信息的同步：

<div align="center">
<img src="/images/posts/ceph/rbd_monitor_recovery.jpg" height="400" width="600">  
</div> 

### 数据服务流程

&emsp;&emsp;所有的初始化动作完成后，Monitor进入ACTIVE状态，即可响应其它节点的读写请求。前面已经说过，对于读请求，所有Monitor均可直接提供数据信息且不涉及内部状态变化；对于写请求，只有Leader能发起变更申请(Peon只能将写请求转发给Leader发请)，在写请求被正式接受之前，Monitor是不能提供该信息的读取服务的。写请求被接受的流程如下：

<div align="center">
<img src="/images/posts/ceph/rbd_monitor_update.jpg" height="400" width="600">  
</div> 


<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【Rados Block Device】九、Monitor原理分析](https://rootw.github.io/2018/04/RBD-Monitor) 
