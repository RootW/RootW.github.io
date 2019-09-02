---
layout: post
title: 【firecracker】CPU与内存
date: 2019-09-01
tags: firecracker
---

### 运行原理

&emsp;&emsp;firecracker虚拟机的CPU与内存功能依赖于硬件辅助虚拟化能力(如intel vt-x特性)：

>* 硬件辅助虚拟化技术为物理CPU增加了一种新的执行模式(Guest Mode/None-Root Mode),在该模式下，CPU中执行的是虚拟机的指令，而非物理机指令。这是一种高效的虚拟指令执行方式，但是在该模式下并不能直接执行所有的虚拟机执令，某些特权指令(如IO指令)的执行会强制将CPU退出到普通模式(Root Mode)交由VMM程序(内核KVM模块/用户态firecracker)进行处理，处理完成后再重新返回到Guest模式执行。

>* 同时，MMU中增加了一级页表(X86架构下叫EPT)，MMU通过该页表完成虚拟物理地址(Guest Physical Address,GPA)到主机物理地址(Host Physical Address, HPA)的转换。也就是说当CPU处于Guest模式时，CPU发出的内存访问请求，MMU需要经过两层转换：首先是完成由虚拟地址到虚拟机物理地址的转换，接着是完成由虚拟机物理地址到真实物理地址的转换。因此可以保证不同虚拟机在内存访问上的隔离性。

<div align="center">                                                             
    <img src="/images/posts/firecracker/cpu.jpg" height="442" width="567">  
</div>

&emsp;&emsp;关于硬件辅助虚拟化技术的详细描述可以参考intel手册和KVM内核模块代码，这里不对此进行深入讨论。firecracker的vCPU线程会对部分退出事件进行处理，我们可以结合代码来进一步讨论其实现原理：

```nohighlight
firecracker/vmm/src/vstate.rs:

pub struct Vcpu {                       // Vcpu结构代表虚拟CPU                          
    …                                                                
    fd: VcpuFd,                         // fd是KVM内核模块为每个vCPU建立的文件句柄，通过该句柄可以向KVM发起vCPU相关的操作命令                    
    id: u8,                             // id是vCPU编号，从0开始            
    io_bus: devices::Bus,               // io_bus是端口总线，所有Legacy设备均挂在该总线下，CPU通过访问不同端口控制这些设备                          
    mmio_bus: Option<devices::Bus>,     // mmio_bus是mmio总线，所有mmio设备均挂在该总线下，CPU通过访问内存指令来控制这些设备，
                                        //     只不过mmio地址空间(0xD0000000~0xFFFFFFFF)与普通内存DRAM地址空间完全独立。这里Option
                                        //     类型是Rust标准库中定义的一种枚举类型，代表对象为空(None)或非空(Some)，目的在于避免
                                        //     使用C语言中的空指针从而引发各种系统错误。每个vCPU对象对应的mmio_bus在初始时为None，
                                        //     后续在vCPU线程进入Guest模式前被设置为正确的mmio总线对象                                      
    …                                                      
}                                                                               

impl Vcpu {                                    // Rust语言中可以为结构体实现各种方法，类似面向对象语言中类的概念
    …
    pub fn run(                                // Vcpu结构中的run方法为每个vCPU线程的主函数                                       
            &mut self,                                                                                             
            …                                                                               
        ) {                                                                                                        
        …                                                                                                                
        while self.run_emulation().is_ok() {}  // 死循环结构调用run_emulation方法，  如果该方法执行出错则退出循环，线程退出                                                       
        …                                                                                                   
    }
    …

    fn run_emulation(&mut self) -> Result<()> {                                 
        match self.fd.run() {                                     // 通过vCPU句柄调用KVM内核暴露的ioctl接口(KVM_RUN)进入内核态，
                                                                  //     借助硬件辅助虚拟化能力进入Guest模式。通常情况下，物理CPU
                                                                  //     开始执行虚拟机指令，只有出现虚拟机指令无法执行或执行异常
                                                                  //     时，物理CPU才会退出到KVM模块进行处理(例如EPT缺页异常)。
                                                                  //     如果KVM模块也无法处 理，会进一步返回到用户态vmm程序，则
                                                                  //     下一条语句才开始执行
            Ok(run) => match run {                                // 如果执行到这里，说明出现了KVM内核也无法处理的退出事件。如果
                                                                  //     返回结果OK，则进一步判断退出事件类型
                VcpuExit::IoIn(addr, data) => {                   // 该分支处理vCPU的端口读操作(IoIn)，addr代表端口地址，data代表将
                                                                  //     取的数据
                    self.io_bus.read(u64::from(addr), data);      // 将端口读操作转发到端口总线io_bus上，其内部根据端口地址搜索对
                                                                  //     应端口设备并调用设备的read接口      
                    …                             
                    Ok(())                                        // run_emulation返回处理成功，处层死循环将再次进入Guest模式
                }                                                               
                VcpuExit::IoOut(addr, data) => {                  // 该分支处理vCPU端口写操作(IoOut)
                    …                                
                    self.io_bus.write(u64::from(addr), data);                   
                    …                            
                    Ok(())                                                      
                }                                                               
                VcpuExit::MmioRead(addr, data) => {               // 该分支处理vCPU MMIO地址读操作(MmioRead)       
                    if let Some(ref mmio_bus) = self.mmio_bus {   // 如果vCPU可访问的mmio总线存在，            
                        mmio_bus.read(addr, data);                // 则将读操作转到到mmio总线上，其内部根据mmio地址搜索对应的mmio设备
                                                                  //         并调用设备的read接口 
                        …                      
                    }                                                           
                    Ok(())                                                      
                }                                                               
                VcpuExit::MmioWrite(addr, data) => {              // 该分支处理vCPU MMIO地址写操作(MmioWrite)        
                    if let Some(ref mmio_bus) = self.mmio_bus {                 
                        …                                                       
                        mmio_bus.write(addr, data);                              
                        …                      
                    }                                                           
                    Ok(())                                                      
                }                                                                
                VcpuExit::Hlt => {                                // 该分支处理Hlt指令，vCPU线程退出                         
                    info!("Received KVM_EXIT_HLT signal");                       
                    Err(Error::VcpuUnhandledKvmExit)                            
                }                                                               
                VcpuExit::Shutdown => {                           // 该分支处理shutdown命令，vCPU线程退出
                    info!("Received KVM_EXIT_SHUTDOWN signal");                 
                    Err(Error::VcpuUnhandledKvmExit)                            
                }  
                …
            }
            …
        }
    }                             
}

```




<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【firecracker】CPU与内存](https://rootw.github.io/2019/09/firecracker-cpu-memory/) 
