---
layout: post
title: 【firecracker】时钟与中断
date: 2019-09-02
tags: firecracker
---

&emsp;&emsp;firecracker虚拟机的时钟与中断系统完全是由KVM模块和硬件实现的，这里仅简要说明其原理，更深入的分析需要结合KVM代码和内核代码进行。

### 时钟系统原理

&emsp;&emsp;时钟系统包含时间源和时钟事件源两部分：
>* 时间源类似生活中的手表，系统通过它可以获知时间，其本质为一个递增的计数器，计数器数值代表当前时间；firecracker中的时钟源有KVM_CLOCK和TSC两种，默认使用KVM_CLOCK；
>* 时种 事件源类似生活中的闹钟，系统通过它可以获得时间到期通知事件，其本质为一个递减的计数器，且当计数器清零时会产生中断通知；firecracker中的时间事件源有PIT和LAPIC_TIMER两种，系统启动过程中使用PIT，一旦系统启动完成后会切换到LAPIC_TIMER。

### 中断系统原理

&emsp;&emsp;X86系统在单核时代是通过两片级联的8259A芯片实现可编程中断控制器，其原理如下图所示。KVM内核也实现了对8259A的模拟，SMP系统在启动初期是采用8259A作为中断控制器。

<div align="center">                                                             
    <img src="/images/posts/firecracker/8259A.jpg" height="303" width="514">  
</div>

&emsp;&emsp;到了SMP多核时代，X86系统采用LAPIC+IOAPIC作为中断控制器系统：每个CPU核都有一个LAPIC，作为本地中断控制器；主板上有一个或多个IOAPIC，外设通过IOAPIC广播中断消息给LAPIC，并由各个LAPIC判断是否需要接收并处理该中断消息，如下图蓝色线条所示。SMP在启动过程中通过MP_Table获知了系统中断控制器信息，之后便可以由8259A切换到LAPIC+IOAPIC。

<div align="center">                                                             
    <img src="/images/posts/firecracker/machine_model.jpg" height="342" width="549">  
</div>

&emsp;&emsp;虽然中断控制器是由KVM模块实现的，但是外设是由用户态VMM程序模拟的，因此KVM模块需要给用户态程序提供触发中断的系统调用接口，即irqfd。它的大致用法如下：

```nohighlight

// initializing irqfd for irq(0)
let device_fd = EventFd::new().unwrap();     // 创建一个新的EventFd句柄
Vm.register_irqfd(device_fd.as_raw_fd(), 0); // 通知KVM模块将该句柄与irq进行关联，这里以0号irq(即时钟中断)为例
// trigger interrupt for irq(0)
device_fd.wirte(1);                          // 通过写EventFd句柄触发一次0号中断，KVM内核会将0号中断通过IOAPIC
                                             // 路由到对应的LAPIC，并向vCPU注入中断
```

### 时钟与中断初始化流程setup_interrupt_controller

```nohighlight
firecracker/vmm/src/lib.rs:

struct Vmm {
   …
}

impl Vmm {
    fn setup_interrupt_controller(&mut self) -> std::result::Result<(), StartMicrovmError> {
        self.vm                                                                     
            .setup_irqchip()                                                         
            .map_err(StartMicrovmError::ConfigureVm)                                
    }
}


firecracker/vmm/src/vstate.rs:

struct Vm {
    …
}

impl Vm {
    pub fn setup_irqchip(&self) -> Result<()> {                                    
        self.fd.create_irq_chip().map_err(Error::VmSetup)?;      // 通知KVM内核模块创建8259A及IOPAIC+LAPIC中断控制器              
        let mut pit_config = kvm_pit_config::default();     

        pit_config.flags = KVM_PIT_SPEAKER_DUMMY;                                   
        self.fd.create_pit2(pit_config).map_err(Error::VmSetup)  // 通知KVM创建PIT时钟事件源                  
    }
}
```

<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【firecracker】时钟与中断](https://rootw.github.io/2019/09/firecracker-interrupt/) 
