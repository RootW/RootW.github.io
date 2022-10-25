---
layout: post
title: 自顶向下设计与实现Virtio
date: 2022-10-25
tags: virtio
---

### 什么是Virtio？为什么需要Virtio？

&emsp;&emsp;Virtio看字面是由Virt(Virtualization)和IO(Input/Output)两部分组成，含义是虚拟化环境下使用的IO协议/设备。早期在虚拟化技术的实现过程中，为了重用现有的设备驱动，虚拟化设备模拟软件一般都是模拟已有物理设备的行为(遵循物理IO协议)，比如通过模拟IDE硬盘可以在虚拟机中完全重用IDE驱动。但是这里引发一个问题，模拟已有的物理设备行为往往导致虚拟化软件性能较低，比如IDE硬件是通过端口(Port)进行数据输入输出的，一段连续的数据需要大量端口操作才能完成。那么有没有可能引入一种新的协议标准，这种协议标准虽然在物理设备中是不存在的，但是可以提升虚拟化场景下设备对数据的输入/输出性能，并且具备一定的通用性，也就是说磁盘、网卡、显卡、键盘、鼠标等等设备都可以使用。

&emsp;&emsp;因此Virtio是在虚拟化场景中引入的一种高性能、通用IO协议，它可以让CPU批量地以队列方式向设备发送请求(Request)和输出数据(Output)，当设备处理完成请求后，也可以让设备批量地以队列方式向CPU通知处理结果(Response)和输入数据(Input)。

<div align="center">                                                             
    <img src="/images/posts/virtio/top.png" height="247" width="180">  
</div>

### 如何逐步实现Virtio？

&emsp;&emsp;从逻辑视角上分析，Virtio顶层所定义的CPU和设备之间批量请求、结果和数据的传输功能是非常基础的能力，无须进一步展开。我们可以在生活中找到非常多类似的例子，比如项目经理通过电话或邮件向项目成员发送了一系列任务，项目成员完成任务后，又通过电话或邮件向项目经理知会完成情况。那么实现Virtio重点就是从物理视角进行展开。

&emsp;&emsp;从物理视角展开分析之前，我们先要了解CPU与设备之间的通信是建立的总线协议之上的，其作用这就好比上面例子中的电话或邮件。以PCI标准总线为例，通过PCI桥可以将设备中的寄存器或存储器映射到CPU可见的内存地址空间中，CPU通过执行对这些地址空间的内存读/写指令就可以完成对设备中寄存器的读/写操作(内存读/写指令会由PCI桥完成，PCI桥根据访存地址决定是发送给物理内存，还是通过PCI总线发送给设备)；反过来，设备也可以通过PCI总线访问物理内存(DMA)或者发送中断通知给CPU(MSI/MSIX)。除了PCI总线，Virtio协议也可以构建在其它总线类型上(如MMIO总线)，它们的作用都是类似的，就是让CPU可以操作设备寄存器、让设备可以操作系统内存并通知CPU有事件发生。

&emsp;&emsp;理解PCI总线机制后，我们就可以设计出物理视角上的项层IO处理流程，这里以单个IO进行示意：

<div align="center">                                                             
    <img src="/images/posts/virtio/IO.png" height="536" width="488">  
</div>

&emsp;&emsp;对于一个IO，其完成过程需要经历以下几步：
>* CPU将IO请求放入内存请求队列中(请求队列就是存放请求的地方，这样可以批量、异步地发送请求)；
>* CPU通过写设备中特定的寄存器唤醒设备并通知它有IO请求待处理；
>* 设备被唤醒后通过DMA操作从内存请求队列中取出IO请求；
>* 设备将IO请求交给内部核心处理逻辑进行处理，比如交给块设备(Blk)通过DMA进行数据存取或者网络设备(Net)进行数据收发；
>* 设备处理完成后将IO响应放入内存响应队列；
>* 设备以中断方式通知CPU有IO响应待处理(表示IO请求处理结果)；
>* CPU在中断上下文中从IO响应队列中取出IO响应进行处理，并最终告知应用程序IO处理结果。

&emsp;&emsp;这里有一个设计选择问题就是请求队列和响应队列为什么要放在系统物理内存中？从实现上说，请求队列和响应队列也可以放到设备的寄存器或存储单元中，这样CPU在放入请求或者设备在放入响应时，就可以一步到位，不用通过内存作为中转。实际上，当放入请求动作对应的数据写入量不大的时候，的确可以提升放入请求动作的性能，但此时CPU是处于Stall状态直到写入操作完成。那么当放入动作对应的数据写入量较大时，CPU就会长时间处于Stall状态，导致CPU计算效率变低。所以Virtio协议在设计时选择将队列置于系统内存中而不是直接放在设备中，而有些协议(如NVMe)选择同时支持两种方式。

&emsp;&emsp;从顶层IO处理流程上看，CPU和设备都是基于内存进行读写操作从而完成请求和响应的传递，那么我们该如何设计内存中队列的数据结构？这是Virtio协议设计中最为关键的展开环节。

&emsp;&emsp;首先，既然是队列，最合适的数据结构是数组，那么数组元素该如何设计？数组元素代表的是每个IO请求、数据或者IO响应，而对于不同类型的设备(磁盘、网卡等等)，请求、数据或响应的内部结构又是不相同的。我们可以观察到无论什么样的请求、数据或响应，从结构本质上看，都是由一段或几段连续内存组成的，每段连续内存由起始地址和长度两个要素决定。这样我们就可以设计一个数组，数组中的元素包含两个成员：起始地址和长度，每个IO可以对应若干数组元素。

&emsp;&emsp;为了重复使用队列内存空间，通常会以环形方式使用数组元素，即已经处理完成的IO对应的数组元素可以用来存放新的IO。但是IO完成的顺序和IO放入的顺序并不完全一致，先取出的IO可能最后才完成，且每个IO对应的元素个数也不相同，这就导致空闲的数组元素并不完全连续。一个可行的解决思路就是采用链表结构来链接各个元素，元素在数组中的位置可以不连续。除了链表，还有没有其它解决思路？如果每个IO就对应一个数组元素，不连续也没啥问题，那就不用链表了，这个思路在Virtio中对应的就是Indirect特性，这里暂不展开分析。

&emsp;&emsp;这样我们就有了一个基本的数组结构，用来表示每个IO对应的请求、数据和响应，这就是Virtio中的Descriptor Table，如下所示(flag的作用后续说明)：

<div align="center">                                                             
    <img src="/images/posts/virtio/descriptor.png" height="430" width="608">  
</div>

&emsp;&emsp;接下来要考虑的问题就是当CPU把一个IO包含的请求、数据、响应对应的内存信息以链表方式填入Descriptor Table中后，那么怎么让设备知道当前有多少个IO并且知道每个IO链表的确切元素？一个解决思路就是设置一个计数变量和一个链表头数组，计数变量用来表明当前共有多少个IO，而链表头数组用来表明每个IO对应的链表头在Descriptor Table中的位置，这就是Virtio中的Available Ring。同样，当IO完成时，设备也需要让CPU知道共有多少个完成的IO和每个IO链表的确切元素，这就是Virtio中的Used Ring，如下所示：

<div align="center">                                                             
    <img src="/images/posts/virtio/avail_used.png" height="438" width="597">  
</div>

&emsp;&emsp;数据结构设计另外一个要考虑的重要问题就是并发操作时的数据同步难题，通常情况当涉及多个CPU同时操作相同数据结构时会采用锁机制进行同步，但是在CPU和设备间无法直接采用锁机制进行同步。这里采用的是无锁队列的设计模式，相同的内存区域只有一方有写入权限，不允许CPU和设备同时写入。Virtio设计中，Descriptor Table和Available Ring只有CPU有写入权限，而Used Ring只有设备有写入权限。

&emsp;&emsp;完成数据结构设计后，我们就可以清楚地看到每个IO步骤对应的实现细节，这里我们以向Virtio-Blk设备发送一个读取命令为例进行说明。

&emsp;&emsp;首先，CPU放入IO请求并通知设备：CPU发起的对Virtio-Blk的读IO操作在内存对应三段内存，分别是head、data buffer和status。head表示操作类型(如读或写)、起始扇区和数据长度，data buffer是存放从设备中读取的数据的内存区域，status表示IO操作结果(0代表成功)。对于读操作来说，head由CPU写入，data buffer和status由设备写入，因此我们在Descriptor Table中申请空闲三项元素，依次填入内存地址和长度，并组成链表。flag标志位
的NEXT标志代表存在下一项，WRITE标志代表由设备写入。然后将Available Ring中的idx由0更新为1，代表有新的IO待处理，并将数组中的第一个元素更新为IO链表头部元素在Descriptor Table中的下标。接着通过总线提供的通知机制，通知设备有IO待处理，例如PCI总线的MMIO机制。过程如下图所示：

<div align="center">                                                             
    <img src="/images/posts/virtio/put_notify.png" height="344" width="760">  
</div>

&emsp;&emsp;其次，设备取出IO请求并处理：设备内部通过last_avail变量记录当前已经处理的IO请求位置，初始为0。设备被唤醒后，通过对比Available Ring中的idx(此时值为1)和last_avail来确定是否有未处理的IO请求，两者不一致则表示有IO请求未处理。然后以last_avail对数组长度取余后的结果为索引去读取Available Ring中数组元素的值，该值代表IO链表头部元素在Descriptor Table中的位置。通过链表头我们就可以从Descriptor Table中找到IO对应的所有内存段，就可以还原完整的IO命令。取出IO请求后设备便将last_avail变量加1，等于Available Ring的idx，表示IO请求已全部取出。取出的IO请求交给设备中的Blk模块执行真正的读取动作，把数据和读取结果通过DMA写入到data buffer和status中。过程如下图所示：

<div align="center">                                                             
    <img src="/images/posts/virtio/get_deal.png" height="283" width="923">  
</div>

&emsp;&emsp;接着，设备放入IO响应并通知CPU：设备完成IO操作后，将完成IO链表头部元素在Descriptor Table中的下标位置写入Used Ring的数组中，并将Used Ring头部idx值由0更新为1，表示有一个IO响应待处理。然后设备通过中断方式通知CPU处理。过程如下图所示：

<div align="center">                                                             
    <img src="/images/posts/virtio/rsp_put.png" height="344" width="761">  
</div>

&emsp;&emsp;最后，CPU取出IO响应并处理：CPU内部通过last_used变量记录当前已经处理的IO响应位置，初始为0。被设备中断后，通过对比Used Ring中的idx(此时值为1)和last_used来确定是否有未处理的IO响应，两者不一致则表示有IO响应未处理。然后以last_used对数组长度取余后的结果为索引去读取Used Ring中数组元素的值，该值代表IO链表头部元素在Descriptor Table中的位置，此时为0。IO响应取出后，CPU将last_used变量加1，然后将IO链表元素进行释放重新作为空闲Descritptor Table项。过程如下图所示：

<div align="center">                                                             
    <img src="/images/posts/virtio/rsp_get.png" height="363" width="721">  
</div>

<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [自顶向下设计与实现Virtio](https://rootw.github.io/2022/10/design-virtio/) 