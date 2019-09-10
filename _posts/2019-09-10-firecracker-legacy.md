---
layout: post
title: 【firecracker】legacy设备
date: 2019-09-10
tags: firecracker
---

&emsp;&emsp;除了virtio-mmio设备，firecracker还实现了串口、键盘等legacy设备，这些设备可以直接由虚拟机内核自带驱动程序驱动，并为虚拟机提供基本的输入输出功能。X86架构下，legacy设备主要通过IO端口进行控制和操作，因此firecracker采用IO总线来组织和管理所有的legacy设备。

### 运行时原理

&emsp;&emsp;和mmio总线类似，legacy设备也以BTree形式组织于devices::Bus下，并且每个设备都需要实现BusDevice trait：

```nohighlight
firecracker/devices/src/legacy/i8042.rs:

pub struct I8042Device {                               // i8042为键盘对象
    …
}

impl BusDevice for I8042Device {                       // 实现端口读写操作                                  
    fn read(&mut self, offset: u64, data: &mut [u8]) {
        …
    }

    fn write(&mut self, offset: u64, data: &[u8]) {
        …
    }
}
```
```nohighlight
firecracker/devices/src/legacy/serial.rs:

pub struct Serial {                                    // 串口对象
    …
}

impl BusDevice for Serial {                            // 实现端口读写操作                                                  
    fn read(&mut self, offset: u64, data: &mut [u8]) {
        …
    }

    fn write(&mut self, offset: u64, data: &[u8]) {
        …
    }
}
```

&emsp;&emsp;回顾前文分析CPU原理时介绍的内容，当虚拟CPU执行端口读写指令时(in/out指令)，会退出到用户态VMM程序进行处理，这里就是firecracker的vCPU线程上下文。此时vCPU线程就会根据端口地址在IO总线上搜索对应的legacy设备，并调用设备对端口的读写操作接口。具体设备的操作方法大家可以对照相关规范文档展开代码分析。

### 初始化

&emsp;&emsp;firecracker使用LegacyDeviceManager来管理所有legacy设备，并通过其下register_devices方法来初始化legacy设备：

```nohighlight
firecracker/vmm/src/lib.rs:
    fn attach_legacy_devices(&mut self) -> std::result::Result<(), StartMicrovmError> {            // legacy设备整体初始化函数
        self.legacy_device_manager                                                                 //  通过LegacyDeviceManager的register_devices来注册legacy设备                                           
            .register_devices()                                                      
            .map_err(StartMicrovmError::LegacyIOBus)?;                              

        self.vm                                                                      
            .get_fd()                                                               
            .register_irqfd(self.legacy_device_manager.com_evt_1_3.as_raw_fd(), 4)                 // 为串口对应的eventfd申请irqfd
            .map_err(|e| {                                                          
                StartMicrovmError::LegacyIOBus(device_manager::legacy::Error::EventFd(e))
            })?;                                                                    
        …                                                                     
        self.vm                                                                     
            .get_fd()                                                               
            .register_irqfd(self.legacy_device_manager.kbd_evt.as_raw_fd(), 1)                     // 为键盘对应的eventfd申请irqfd
            .map_err(|e| StartMicrovmError::LegacyIOBus(device_manager::legacy::Error::EventFd(e)))
}


firecracker/vmm/src/device_manager/legacy.rs:

pub struct LegacyDeviceManager {                           // 所有legacy设备管理器                                             
    pub io_bus: devices::Bus,                              // IO总线，以BTree组织legacy设备
    pub stdio_serial: Arc<Mutex<devices::legacy::Serial>>, // 串口对象 
    pub i8042: Arc<Mutex<devices::legacy::I8042Device>>,   // 键盘对象                     

    pub com_evt_1_3: EventFd,                              // 串口1、3对应的eventfd                                                 
    pub com_evt_2_4: EventFd,                              // 串口2、4对应的eventfd
    pub kbd_evt: EventFd,                                  // 键盘对应的eventfd
    pub stdin_handle: io::Stdin,                           // 标准输入
}

impl LegacyDeviceManager {
    …                                                            
    pub fn register_devices(&mut self) -> Result<()> {                          
        self.io_bus                                        // 将串口对象添加到端口地址0x3F8~0x3FF，
            .insert(self.stdio_serial.clone(), 0x3f8, 0x8) // 由规范定义               
            .map_err(Error::BusError)?;                                         
        …                                                              
        self.io_bus                                        // 将键盘对象添加到端口地址0x60~0x64 ，                                                     
            .insert(self.i8042.clone(), 0x060, 0x5)        // 由规范定义            
            .map_err(Error::BusError)?;    
        Ok(())                                                                  
    }                                                                           
}
```
<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【firecracker】legacy设备](https://rootw.github.io/2019/09/firecracker-legacy/) 
