---
layout: post
title: 自顶向下设计与实现Virtio
date: 2022-10-25
tags: virtio
---

### 什么是Virtio？为什么需要Virtio？

&emsp;&emsp;Virtio看字面是由Virt(Virtualization)和IO(Input/Output)两部分组成，含义是虚拟化环境下使用的IO协议/设备。早期在虚拟化技术的实现过程中，为了重用现有的设备驱动，虚拟化设备模拟软件一般都是模拟已有物理设备的行为(遵循物理IO协议)，比如通过模拟IDE硬盘可以在虚拟机中完全重用IDE驱动。但是这里引发一个问题，模拟已有的物理设备行为往往导致虚拟化软件性能较低，比如IDE硬件是通过端口(Port)进行数据输入输出的，一段连续的数据需要大量端口操作才能完成。那么有没有可能引入一种新的协议标准，这种协议标准虽然在物理设备中是不存在的，但是可以提升虚拟化场景下设备对数据的输入/输出性能，并且具备一定的通用性，也就是说磁盘、网卡、显卡、键盘、鼠标等等设备都可以使用。

&emsp;&emsp;因此Virtio是在虚拟化场景中引入的一种高性能、通用IO协议，它可以让CPU批量地向设备发送请求(Request)和输出数据(Output)，当设备处理完成请求后，也可以让设备批量地向CPU通知结果(Response)和输入数据(Input)。

<div align="center">                                                             
    <img src="/images/posts/virtio/top.png" height="247" width="180">  
</div>

<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [自顶向下设计与实现Virtio](https://rootw.github.io/2022/10/design-virtio/) 