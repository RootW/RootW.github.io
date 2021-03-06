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

&emsp;&emsp;通过上述代码可以看出，CPU与内存的虚拟化功能基本由硬件实现，而软件主要配合硬件完成对各种退出事件的处理：KVM内核模块实现了对EPT表缺页异常的处理；firecracker实现了对端口读写及mmio空间读写操作的处理。

### 初始化流程

&emsp;&emsp;尽管虚拟CPU和内存在运行过程中无须firecracker有过多干涉，但是它们的初始化动作是完全由friecracker实现的，因此我们需要了解firecracker是如何实现CPU和内存初始化的。

#### **一、简要KVM使用例程**

&emsp;&emsp;在我们正式分析firecracker的初始化代码之前，我们有必要先了解一下KVM模块的使用方法，因为firecracker是基于KVM模块构建的。Rust语言环境下有一个名叫kvm-ioctls的库，它实现了对KVM内核ioctl系统调用的封装，我们可以基于这个库实现一个简单的KVM虚拟机程序：启动一个单核，普通内存地址范围0x1000~0x5000的虚拟机，虚拟机进入Guest模式后执行一段简单的端口操作与MMIO空间操作指令后立刻停机。

```nohighlight

fn main() {
    …
    let kvm = Kvm::new().unwrap();      // 第一步：创建一个kvm对象，对应字符设备/dev/kvm

    let vm = kvm.create_vm().unwrap();  // 第二步：通过kvm对象创建一个虚拟机对象vm
                                        // 第三步：初始化虚拟机普通内存
    let mem_size = 0x4000;              //      设置普通内存大小为0x4000                                                  
    let guest_addr: u64 = 0x1000;       //      设置普通内存起始地址为0x1000，意味在0x1000~0x5000之外均为MMIO地址空间 
    let load_addr: *mut u8 = unsafe {   //      通过mmap系统调用映射一段大小为0x4000的区间，虚拟起始地址记录到load_addr中 
        libc::mmap(                                                         
            null_mut(),                                                     
            mem_size,                                                       
            libc::PROT_READ | libc::PROT_WRITE,                             
            libc::MAP_ANONYMOUS | libc::MAP_SHARED
                | libc::MAP_NORESERVE,   
            -1,                                                             
            0,                                                               
        ) as *mut u8                                                        
    };                                                                      

    let slot = 0;                                  //     设置区间号为零，并将之前映射的区间信息传递给KVM        
    let mem_region = kvm_userspace_memory_region { //     该对象包含所有需要传递给KVM的信息                       
        slot,                                      //     当前区间编号，即0
        guest_phys_addr: guest_addr,               //     虚拟机起始物理地址，即0x1000        
        memory_size: mem_size as u64,              //     虚拟机普通内存区间大小，即0x4000        
        userspace_addr: load_addr as u64,          //     普通内存区间映射到主机的虚拟地址           
        flags: KVM_MEM_LOG_DIRTY_PAGES,            //     区间标志，通知KVM对该区间进行脏页跟踪                    
    };                                                                         
    unsafe { vm.set_user_memory_region(mem_region).unwrap() };    // 通过虚拟机对象vm将普通内存信息传递给KVM

    let x86_code = [                                              // 设置虚拟机内将执行的指令    
        0xba, 0xf8, 0x03, /* mov $0x3f8, %dx */                   //     端口寄存器dx设为0x3f8，对应串口设备
        0x00, 0xd8, /* add %bl, %al */                            //     将bl寄存器值 加到al寄存器中
        0x04, b'0', /* add $'0', %al */                           //     将字符'0'的值加到al寄存器中
        0xee, /* out %al, %dx */                                  //     向端口输出al寄存器的值
        0xec, /* in %dx, %al */                                   //     从端口读入值到al寄存器
        0xc6, 0x06, 0x00, 0x80, 0x00,                                                  
            /* movl $0, (0x8000); This generates a MMIO Write.*/  //     将零写到0x8000地址，位于MMIO地址空间
        0x8a, 0x16, 0x00, 0x80,
            /* movl (0x8000), %dl; This generates a MMIO Read.*/  //     读取0x8000地址值到dl寄存器
        0xf4, /* hlt */                                           //     执行停机指令
    ];    

    unsafe {                                                      // 将上述指令内容写入到虚拟机普通内存区间中
        let mut slice = slice::from_raw_parts_mut(load_addr, mem_size);     
        slice.write(&x86_code).unwrap();                                    
    }

    let vcpu_fd = vm.create_vcpu(0).unwrap();                     // 第四步：通过虚拟机对象创建一个vCPU对象
                                                                  // 第五步：初始化vCPU对象中各个寄存器的值
    let mut vcpu_sregs = vcpu_fd.get_sregs().unwrap();            //     获取各个段寄存器的默认值
    vcpu_sregs.cs.base = 0;                                       //     将代码段选择子cs对应的基地址设为0
    vcpu_sregs.cs.selector = 0;                                   //     将代码段选择子cs设为0
    vcpu_fd.set_sregs(&vcpu_sregs).unwrap();                      //     将设置后的段寄存器值传递给KVM

    let mut vcpu_regs = vcpu_fd.get_regs().unwrap();              //     获取通用寄存器的默认值              
    vcpu_regs.rip = guest_addr;                                   //     将指令指针IP设为0x1000
    vcpu_regs.rax = 2;                                            //     将rax寄存器初始化为2 
    vcpu_regs.rbx = 3;                                            //     将rbx寄存器初始化为3；对照上述指令，最后串口将输出字符'5'
    vcpu_regs.rflags = 2;                                         //     将标志寄存器设为2，因为位1为保留位，值须为1
    vcpu_fd.set_regs(&vcpu_regs).unwrap();                        //     将设置后的通用寄存器值传递给KVM

    loop {                                                        // 第六步：通过vCPU对象以死循环方式不断进入Guest模式
        match vcpu_fd.run().expect("run failed") {                          
            VcpuExit::IoIn(addr, data) => {                                 
                println!(                                                   
                    "Received an I/O in exit. Address: {:#x}. Data: {:#x}", 
                    addr,                                                   
                    data[0],                                                 
                );                                                          
            }
            VcpuExit::IoOut(addr, data) => {
                …
            }
            VcpuExit::MmioRead(addr, data) => {
                …
            }
            VcpuExit::MmioWrite(addr, data) => {
                …
            }
            VcpuExit::Hlt => {
                …
            }
            r => panic!("Unexpected exit reason: {:?}", r),
        } // end of match
    } // end of loop
}
```

&emsp;&emsp;通过上述例程，我们看到只需要简单的六大步，便可以通过KVM模块创建一个虚拟机。在掌握了KVM的基础原理后，我们再来看看firecracker中的内存和CPU初始化流程。

#### **二、虚拟机内存初始化init_guest_memory**

&emsp;&emsp;firecracker对虚拟机内存的初始化动作在init_guest_memory函数中完成，代码原理如下：

```nohighlight
firecracker/vmm/src/lib.rs

struct Vmm {                            // vmm全局对象，包含当前虚拟机的所有全局信息
    kvm: KvmContext,                    // kvm上下文，包含kvm对象
    vm_config: VmConfig,                // 虚拟机配置，如CPU数和内存大小等
    guest_memory: Option<GuestMemory>,  // 虚拟机内存对象，初始时为None
    vm: Vm,                             // 虚拟机对象vm
    …
}

impl Vmm {
    fn init_guest_memory(&mut self) -> std::result::Result<(), StartMicrovmError> {
        let mem_size = self                                             // 从虚拟机配置vm_config中获取内存大小并转化为字节单位                                                         
            .vm_config                                                              
            .mem_size_mib                                                           
            .ok_or(StartMicrovmError::GuestMemory(                                   
                memory_model::GuestMemoryError::MemoryNotInitialized,               
            ))?                                                                     
            << 20;                                                                   
        let arch_mem_regions = arch::arch_memory_regions(mem_size);     // 根据不同体系架构，划分内存区间。以X86为例，虚拟机普通
                                                                        //     内存将分布在两个区间：0x0~0xD0000000和0xFFFFFFFF~X。
                                                                        //     地址区间0xD0000000~0xFFFFFFFF为MMIO空间。   
        self.guest_memory =                                                         
            Some(GuestMemory::new(&arch_mem_regions)                    // 通过mmap系统调用将各个区间映射到用户空间，参见KVM例程
            .map_err(StartMicrovmError::GuestMemory)?);
        self.vm                                                                     
            .memory_init(                                               // 通过vm对象将内存区间信息传递给KVM，参见KVM例程                                                 
                self.guest_memory                                                   
                    .clone()                                                         
                    .ok_or(StartMicrovmError::GuestMemory(                          
                        memory_model::GuestMemoryError::MemoryNotInitialized,   
                    ))?,                                                             
                &self.kvm,                                                          
            )                                                                       
            .map_err(StartMicrovmError::ConfigureVm)?;                              
        Ok(())                                                                      
    }
    …
}
```

#### **三、虚拟机CPU初始化create_vcpus和start_vcpus**

&emsp;&emsp;firecracker通过create_vcpus创建虚拟CPU，并通过start_vcpus启动CPU，代码原理如下：

```nohighlight
firecracker/vmm/src/lib.rs

struct Vmm {
    …
    kvm: KvmContext,                                // kvm上下文，包含kvm对象
    vcpus_handles: Vec<thread::JoinHandle<()>>,     // vCPU线程句柄数组，每个vCPU线程对应一个句柄
    vm: Vm,                                         // 虚拟机对象vm
    mmio_device_manager: Option<MMIODeviceManager>, // mmio设备管理器，包含mmio总线及其上的所有mmio设备
    legacy_device_manager: LegacyDeviceManager,     // legacy设备管理器，包含legacy总线及其上的所有legacy设备
    …
}

impl Vmm {
    …
    fn create_vcpus(                              // 创建所有的虚拟CPU                                                       
        &mut self,                                                              
        entry_addr: GuestAddress,                 // 参数entry_addr代表加载虚拟机内核的目的地址，
                                                  //      也是虚拟机进入Guest模式执行的第一条指令位置                                            
        request_ts: TimestampUs,                                                 
    ) -> std::result::Result<Vec<Vcpu>, StartMicrovmError> {                    
        let vcpu_count = self                                                      // 从虚拟机配置vm_config中读取虚拟CPU个数                                                  
            .vm_config                                                          
            .vcpu_count                                                         
            .ok_or(StartMicrovmError::VcpusNotConfigured)?;                     
        let mut vcpus = Vec::with_capacity(vcpu_count as usize);                   // 根据vCPU个数创建向量集vcpus           

        for cpu_id in 0..vcpu_count {                                              // 循环创建vcpu_count个vCPU                                           
            let io_bus = self.legacy_device_manager.io_bus.clone();                //     克隆legacy总线
            let mut vcpu = Vcpu::new(cpu_id, &self.vm, io_bus, request_ts.clone()) //     通过KVM创建一个vCPU
                .map_err(StartMicrovmError::Vcpu)?;                             
            vcpu.configure(&self.vm_config, entry_addr, &self.vm)                  //     配置vCPU寄存器信息，包括段寄存器、通用寄存器和系统寄存器
                                                                                   //           以及虚拟机内部页表。第一个vCPU(BSP)开始运行时即处于分
                                                                                   //           页模式下              
            .map_err(StartMicrovmError::VcpuConfigure)?;                           //  
            vcpus.push(vcpu);                                                      //      将vcpu添加到向量集中
        }                                                                        
        Ok(vcpus)                                                               
    }

    fn start_vcpus(&mut self, mut vcpus: Vec<Vcpu>)
            -> std::result::Result<(), StartMicrovmError> {                        // 启动所有的vCPU
        …                                                                                                     
        for cpu_id in (0..vcpu_count).rev() {                                      // 循环启动vCPU                                  
            …                                   
            let mut vcpu = vcpus.pop().unwrap();                                   //       依次取出vcpu                             

            if let Some(ref mmio_device_manager) = self.mmio_device_manager {      //       如果存在mmio总线，则克隆总线对象
                vcpu.set_mmio_bus(mmio_device_manager.bus.clone());             
            }                                                                   
            …                            
            self.vcpus_handles.push(                                            
                thread::Builder::new()                                             //       启动一个名为fc_vcpu的线程，执行vcpu的run函数，
                                                                                   //       参考运行原理代码分析                                    
                    .name(format!("fc_vcpu{}", cpu_id))                         
                    .spawn(move || {                                            
                        vcpu.run(…);
                    })                                                          
                    .map_err(StartMicrovmError::VcpuSpawn)?,                    
            );                                                                   
        }                                                                       
        …                                                                                                                
        Ok(())                                                                   
    }                                  
}
```

&emsp;&emsp;至此，firecracker虚拟机CPU与内存子系统已经分析完成，简单总结一下：虚拟机CPU和内存的功能基本上是由硬件和KVM模块实现的，firecracker主要对端口和MMIO操作进行了处理。在后续的MMIO设备和legacy设备分析中，我们将深入讨论端口及MMIO空间的处理流程。

<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【firecracker】CPU与内存](https://rootw.github.io/2019/09/firecracker-cpu-memory/) 
