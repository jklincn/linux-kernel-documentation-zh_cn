> 原文：[PCI Bus Subsystem  — The Linux Kernel documentation](https://docs.kernel.org/PCI/index.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.15
>
> 修订：
>
> **已知问题：二级目录需要右键-在新标签页中打开链接才能正确跳转**

# PCI 总线子系统

- [1. 如何编写 Linux PCI 驱动程序](pci.md)
  - [1.1. PCI 驱动程序的结构](pci.md#11-pci-驱动程序的结构)
  - [1.2. pci_register_driver() 调用](pci.md#12-pci_register_driver-调用)
  - [1.3. 如何手动查找 PCI 设备](pci.md#13-如何手动查找-PCI-设备)
  - [1.4. 设备初始化步骤](pci.md#14-设备初始化步骤)
  - [1.5. PCI 设备关闭](pci.md#15-PCI-设备关闭)
  - [1.6. 如何访问 PCI 配置空间](pci.md#16-如何访问-PCI-配置空间)
  - [1.7. 其他有趣的函数](pci.md#17-其他有趣的函数)
  - [1.8. 杂项提示](pci.md#18-杂项提示)
  - [1.9. 供应商和设备ID](pci.md#19-供应商和设备ID)
  - [1.10. 过时的功能](pci.md#110-过时的功能)
  - [1.11. MMIO 空间和写入过账](pci.md#111-MMIO-空间和写入过账)
  
- [2. PCI Express 端口总线驱动程序指南](pciebus-howto.md)
  - [2.1. 关于本指南](pciebus-howto.md#21-关于本指南)
  - [2.2. 什么是 PCI Express 端口总线驱动程序](pciebus-howto.md#22-什么是-pci-express-端口总线驱动程序)
  - [2.3. 为什么使用 PCI Express 端口总线驱动程序](pciebus-howto.md#23-为什么使用-pci-express-端口总线驱动程序)
  - [2.4. 配置 PCI Express 端口总线驱动程序与服务驱动程序](pciebus-howto.md#24-配置-pci-express-端口总线驱动程序与服务驱动程序)
  - [2.5. 可能的资源冲突](pciebus-howto.md#25-可能的资源冲突)
  
- [3. PCI Express I/O 虚拟化指南](pci-iov-howto.md)
  - [3.1. 概述](pci-iov-howto.md#31-概述)
  - [3.2. 用户指南](pci-iov-howto.md#32-用户指南)
  - [3.3. 开发者指南](pci-iov-howto.md#32-开发者指南)
  
- [4. MSI 驱动程序指南](msi-howto.md)
  - [4.1. 关于本指南](msi-howto.md#41-关于本指南)
  - [4.2. 什么是 MSI](msi-howto.md#42-什么是-msi)
  - [4.3. 为什么使用 MSI](msi-howto.md#43-为什么使用-msi)
  - [4.4. 如何使用 MSI](msi-howto.md#44-如何使用-msi)
  - [4.5. MSI 异常情况](msi-howto.md#45-msi-异常情况)
  - [4.6. 设备驱动程序 MSI(-X) API 列表](msi-howto.md#46-设备驱动程序-msi-x-api-列表)
  
- [5. 通过 sysfs 访问 PCI 设备资源](sysfs-pci.md)
  - [5.1. 通过 sysfs 访问传统资源](sysfs-pci.md#51-通过-sysfs-访问传统资源)
  - [5.2. 支持新平台上的 PCI 访问](sysfs-pci.md#52-支持新平台上的-pci-访问)
  
- 6.PCI 主桥的 ACPI 注意事项

- 7.PCI 错误恢复
  - 7.1. 详细设计
  
- 8.PCI Express 高级错误报告驱动程序指南

  - 8.1. 概述
  - 8.2. 用户指南
  - 8.3. 开发者指南
  - 8.4. 软件错误注入

- 9.PCI 端点框架

  - 9.1. 介绍
  - 9.2. PCI 端点核心
  - 9.3. 使用 CONFIGFS 配置 PCI 端点
  - 9.4. PCI 测试功能
  - 9.5. PCI 测试用户指南
  - 9.6. PCI 非透明桥（Non-Transparent Bridge, NTB）功能
  - 9.7. PCI NTB 端点功能（Endpoint Function, EPF）用户指南
  - 9.8. PCI vNTB 功能
  - 9.9. PCI 非透明桥功能
  - 9.10. PCI 测试端点功能
  - 9.11. PCI NTB 端点功能

- 10.启动终端

  - 10.1. 概述
  - 10.2. 问题
  - 10.3. 状况
  - 10.4. 受影响的芯片组
  - 10.5. 缓解措施
  - 10.6. 更多文档

