---
layout: post
title: 【firecracker】系统启动与epoll事件循环
date: 2019-09-10
tags: firecracker
---

&emsp;&emsp;在分析完firecracker虚拟机中各个子系统的运行时原理和初始化流程后，最后我们整体分析一下firecracker的系统启动过程，并重点分析IO线程(fc_vmm)所采用的epoll事件循环框架。

&emsp;&emsp;回顾首文中介绍的firecracker进程的线程模型，下图展示了线程间的事件通知关系：


<div align="center">                                                             
    <img src="/images/posts/firecracker/thread_com.jpg" height="155" width="757">  
</div>

&emsp;&emsp;firecracker进程的主线程在启动完成后会成为API Server Thread，它通过进程间socket句柄与外部HTTP Client进行通信，接收外部程序发送的配置或控制命令(回顾一下使用方法)；API Server Thread收到外部命令后通过api_event_fd向IO Thread发送请求，由IO Thread完成实际的处理动作；当虚拟机正常启动后，IO Thread主要通过io_event_fd接受来自虚拟CPU的IO请求，并在完成IO请求后通过irq_fd通知CPU处理结果。

&emsp;&emsp;接下来，我们从main函数开始，逐步分析一下系统的启动过程。

```nohighlight
firecracker/src/main.rs:

fn main() {
    …
    let shared_info = Arc::new(RwLock::new(InstanceInfo {      // InstanceInfo对象是主线程和fc_vmm线程之间的共享对象
                                                               //     Arc是Rust标准库提供的原子引用类型，跟踪多个线程对相同
                                                               //     对象的引用计数；RwLock是Rust标准库提供的读写锁                      
        state: InstanceState::Uninitialized,                                    
        id: instance_id,                                                        
        vmm_version: crate_version!().to_string(),                               
    }));                                                                        
    let mmds_info = MMDS.clone();                                               
    let (to_vmm, from_api) = channel();                        // 该channel对象用于在主线程和fc_vmm线程之间传递对象
                                                               //     to_vmm为发送端对象，在主线程中使用；from_api为接收
                                                               //     端，在fc_vmm线程中使用                    
    let server =                                                                
        ApiServer::new(mmds_info, shared_info.clone(), to_vmm) // 创建一个ApiServer对象，用于接收外部程序的HTTP请求
        .expect("Cannot create API server");

    let api_event_fd = server                                  // 申请一个event_fd，用于主线程和fc_vmm线程之间的事件通知
        .get_event_fd_clone()                                                   
        .expect("Cannot clone API eventFD.");                                   

    let _vmm_thread_handle =                                                    
        vmm::start_vmm_thread(                                 // 正式拉起fc_vmm线程，下文将进一步分析
            shared_info,
            api_event_fd,
            from_api,
            seccomp_level);

    match server.bind_and_run(                                 // 主线程开始作为API Server线程，监听socket请求
        bind_path, start_time_us,
        start_time_cpu_us, seccomp_level) {
        …                                                                      
    }                                                                           
}


firecracker/vmm/src/lib.rs:

pub fn start_vmm_thread(                                                        
    api_shared_info: Arc<RwLock<InstanceInfo>>,                                 
    api_event_fd: EventFd,                                                       
    from_api: Receiver<Box<VmmAction>>,                                         
    seccomp_level: u32,                                                         
    ) -> thread::JoinHandle<()> {                                                    
    thread::Builder::new()                                                      
        .name("fc_vmm".to_string())                                                           // 将新线程取名为fc_vmm                                     
        .spawn(move || {                                                                      // 创建新线程，参数为新线程入口函数(闭包)
            let mut vmm = Vmm::new(api_shared_info, api_event_fd, from_api, seccomp_level)    // 新线程首先创建全局Vmm对象
                .expect("Cannot create VMM");                                   
            match vmm.run_control() {                                                         // 进入epoll事件循环
                …                                                              
            }                                                                   
        })                                                                       
        .expect("VMM thread spawn failed.")                                     
}

struct Vmm {                                            // Vmm全局对象
    kvm: KvmContext,                                    // KVM操作上下文

    vm_config: VmConfig,                                // 虚拟机配置，如CPU数、内存大小等
    shared_info: Arc<RwLock<InstanceInfo>>,             // API Server线程与fc_vmm线程共享对象            

    // Guest VM core resources.                                                  
    guest_memory: Option<GuestMemory>,                  // 虚拟机内存对象              
    kernel_config: Option<KernelConfig>,                // 内核配置，如启动命令行参数     
    vcpus_handles: Vec<thread::JoinHandle<()>>,         // vCPU线程返回句柄数组            
    exit_evt: Option<EpollEvent<EventFd>>,              // 虚拟机退出事件eventfd        
    vm: Vm,                                             // 虚拟机对象

    // Guest VM devices.                                                        
    mmio_device_manager: Option<MMIODeviceManager>,     // mmio总线管理器                             
    legacy_device_manager: LegacyDeviceManager,         // legacy总线管理器                

    // Device configurations.                                                    
    block_device_configs: BlockDeviceConfigs,           // 后端存储块设备配置，供virtio-blk使用      
    network_interface_configs: NetworkInterfaceConfigs, // 后端网络接口配置，供virtio-net使用                   
    …                                    

    epoll_context: EpollContext,                        // epoll事件循环上下文

    // API resources.                                                           
    api_event: EpollEvent<EventFd>,                     // 接收API Server线程通知的eventfd
    from_api: Receiver<Box<VmmAction>>,                 // channel接收方，可接收来向API Server线程对象         
    …                                                       
}

impl Vmm {                                                                      
    fn new(                                                                      
        api_shared_info: Arc<RwLock<InstanceInfo>>,                             
        api_event_fd: EventFd,                                                  
        from_api: Receiver<Box<VmmAction>>,                                      
        seccomp_level: u32,                                                     
    ) -> Result<Self> {                                                         
        let mut epoll_context = EpollContext::new()?;                  // 初始化epoll上下文              
        let api_event = epoll_context                                           
            .add_event(api_event_fd, EpollDispatch::VmmActionRequest)  // 将api_event_fd添加到epoll上下，可监听该句柄          
            .expect("Cannot add API eventfd to epoll.");                                     

        let block_device_configs = BlockDeviceConfigs::new();          // 初始化存储配置对象           
        let kvm = KvmContext::new()?;                                  // 创建KVM上下文
        let vm = Vm::new(kvm.fd()).map_err(Error::Vm)?;                // 创建虚拟机对象

        Ok(Vmm {                                                                
            kvm,                                                                
            vm_config: VmConfig::default(),                                     
            shared_info: api_shared_info,                                       
            guest_memory: None,                                                 
            kernel_config: None,                                                 
            vcpus_handles: vec![],                                              
            exit_evt: None,                                                     
            vm,                                                                  
            mmio_device_manager: None,                                          
            legacy_device_manager: LegacyDeviceManager::new().map_err(Error::CreateLegacyDevice)?,
            block_device_configs,                                                
            network_interface_configs: NetworkInterfaceConfigs::new(),          
            …
            epoll_context,                                                      
            api_event,                                                           
            from_api,                                                           
            …                                                       
        })                                                                       
    } 

    fn run_control(&mut self) -> Result<()> {                                                       // 事件循环框架                                
        const EPOLL_EVENTS_LEN: usize = 100;                                     

        let mut events = vec![epoll::Event::new(epoll::Events::empty(), 0); EPOLL_EVENTS_LEN];      // 创建一个events数组，用于接收事件

        let epoll_raw_fd = self.epoll_context.epoll_raw_fd;                     

        'poll: loop {                                                                               // 循环入口                                                     
            let num_events = epoll::wait(epoll_raw_fd, -1, &mut events[..]).map_err(Error::Poll)?;  // 通过epoll获知已经发生的事件

            for event in events.iter().take(num_events) {                                           // 针对已经发生的事件依次进行处理                       
                let dispatch_idx = event.data as usize;                                             // 获知事件dispatch_idx，注册事件时传入                       
                let evset = match epoll::Events::from_bits(event.events) {                          // 获知具体事件，如POLLIN     
                    …                                                            
            };                                                               

            if let Some(dispatch_type) = self.epoll_context.dispatch_table[dispatch_idx] {          // 根据disptach_idx找到事件类型，注册时填入
                match dispatch_type {                                       
                    EpollDispatch::Exit => {                                                        // 第一类别，退出；VCPU线程退出时产生
                        …               
                    }                                                       
                    EpollDispatch::Stdin => {                                                       // 第二类型，标准输入；使能串口使注册，接
                                                                                                    // 收标准输入并作为虚拟机串口的输入                              
                        …                     
                    }                                                       
                    EpollDispatch::DeviceHandler(device_idx, device_token) => {                     // 第三类型，virtio设备IO处理；VCPU通过io_event_fd产生     
                        match self                                          
                            .epoll_context                                  
                            .get_device_handler_by_handler_id(device_idx)                            // 首次处理时通过channel获取EpollHandler对象，回顾virtio
                        {                                                                            // 设备的activate流程
                            Ok(handler) => match handler.handle_event(device_token, evset) {         // 调用EpollHandler对象的handle_event函数
                                …                                    
                            },                                              
                            …                                               
                        }                                                    
                    }                                               
                    EpollDispatch::VmmActionRequest => {                                             // 第四类型，管理动作；来作API Server线程          
                        self.api_event.fd.read().map_err(Error::EventFd)?;  
                        self.run_vmm_action().unwrap_or_else(|_| {                                   // 调用run_vmm_action     
                            …
                        });                                                 
                    }                                                       
                    …                                                      
                }                                                           
            }                                                               
        }                                                                    
    }                                                                       
}
```

&emsp;&emsp;firecracker启动初期，fc_vmm线程仅对api_event_fd进行监听，即仅能对第四类型事件进行处理。回顾首篇对firecracker使用流程的介绍，我们通过curl工具对firecracker进行kernel、rootfs和虚拟机的配置后，最后通过InstanceStart命令启动虚拟机。这里的配置和启动命令最后都交由fc_vmm的run_vmm_action函数进行处理：

```nohighlight

    fn run_vmm_action(&mut self) -> Result<()> {                                                             
        let request = match self.from_api.try_recv() {                                    // 从API Server线程接收用户请求                      
            …                                                                 
        };                                                                      

        match request {                                                                    // 根据用户请求类别分别进行处理                                                      
            VmmAction::ConfigureBootSource(boot_source_body, sender) => {                  // 内核启动参数配置     
                Vmm::send_response(                                             
                    self.configure_boot_source(                                 
                        boot_source_body.kernel_image_path,                     
                        boot_source_body.boot_args,                             
                    ),                                                          
                    sender,                                                      
                ); 

            }                                                                   
            …                                                                
            VmmAction::InsertBlockDevice(block_device_config, sender) => {                 // 配置存储块设备
                Vmm::send_response(self.insert_block_device(block_device_config), sender);
            }                                                                    
            VmmAction::InsertNetworkDevice(netif_body, sender) => {                        // 配置网络接口设备
                Vmm::send_response(self.insert_net_device(netif_body), sender); 
            }                                                                    
            …                                                                 
            VmmAction::StartMicroVm(sender) => {                                           // 配置完成后，启动一个虚拟机
                Vmm::send_response(self.start_microvm(), sender);               
            }                                                                                                                                    
            VmmAction::SetVmConfiguration(machine_config_body, sender) => {                // 配置虚拟机CPU和内存等    
                Vmm::send_response(self.set_vm_configuration(machine_config_body), sender);
            }                                                                   
            …                                                                    
        };                                                                      
       Ok(())                                                                  
    }
```

&emsp;&emsp;最后，我们来看一下firecracker虚拟机的启动过程，对各个子系统初始化进行一些串联：

```nohighlight
    fn start_microvm(&mut self) -> std::result::Result<VmmData, VmmActionError> {
        …         
        self.shared_info                                                                       
            .write()                                               // 对共享对象加写锁
            .expect("Failed to start microVM because shared info couldn't be written due to poisoned lock")
            .state = InstanceState::Starting;                      // 将虚拟机状态设为Starting，完成后自动解锁(Rust语言特性)

        self.init_guest_memory()?;                                 // 虚拟内存初始化，参考CPU与内存部分

        let vcpus;                                                              

        #[cfg(target_arch = "x86_64")]                                          
        {                                                                       
            self.setup_interrupt_controller()?;                   // 中断控制器初始化，参考时钟与中断部分       
            self.attach_virtio_devices()?;                        // 添加virito-blk/net，内部将调用register_virtio_device，参考virtio设备部分
            self.attach_legacy_devices()?;                        // legacy设备初始化，参考legacy设备部分

            let entry_addr = self.load_kernel()?;                 // 加载ELF内核到entry_addr
            vcpus = self.create_vcpus(entry_addr, request_ts)?;   // 创建VCPU，参考CPU与内存部分             
        }
        …                                                               
        self.configure_system()?;                                 // 配置系统，主要是生成mptable和引导数据头部

        self.register_events()?;                                  // 向epoll事件循环注册退出事件和标准输入事件

        self.start_vcpus(vcpus)?;                                 // 启动虚拟CPU，参考CPU与内存部分                                               

        self.shared_info                                                         
            .write()                                              // 重新对共享对象加写锁
            .expect("Failed to start microVM because shared info couldn't be written due to poisoned lock")
            .state = InstanceState::Running;                      // 将虚拟机状态设为Running，代表虚拟机已正常运行!!!                                                                   
        …                                                                                                                                          
        Ok(VmmData::Empty)                                                      
    }                
```
<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【firecracker】系统启动与epoll事件循环](https://rootw.github.io/2019/09/firecracker-startvm/) 
