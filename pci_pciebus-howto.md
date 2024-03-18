> 原文：[2. The PCI Express Port Bus Driver Guide HOWTO — The Linux Kernel documentation](https://docs.kernel.org/PCI/pciebus-howto.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.18
>
> 修订：

# 2. PCI Express 端口总线驱动程序指南 

## 2.1. 关于本指南

本指南介绍了 PCI Express 端口总线驱动程序的基本知识，并提供了有关如何使服务驱动程序能够使用 PCI Express 端口总线驱动程序进行注册/注销的信息。

## 2.2. 什么是 PCI Express 端口总线驱动程序

PCI Express 端口是一种逻辑 PCI-PCI 桥结构。PCI Express 端口有两种类型：根端口和交换机端口。根端口从 PCI Express 根复合体发起 PCI Express 链路，交换机端口将 PCI Express 链路连接到内部的逻辑 PCI 总线。如果一个交换机端口的次级总线代表交换机内部的路由逻辑，那么这个端口被称为上行端口。交换机的下行端口从交换机内部路由总线桥接到 PCI Express 交换机中代表下游 PCI Express 链路的总线。

> 译者注：这边关于上/下行端口的定义有点难理解，简单来说，靠近根复合体的是上行端口，连接设备的是下行端口。原文：The Switch Port, which has its secondary bus representing the switch's internal routing logic, is called the switch's Upstream Port. The switch's Downstream Port is bridging from switch's internal routing bus to a bus representing the downstream PCI Express link from the PCI Express Switch.

一个 PCI Express 端口最多可提供四种不同的功能，在本文档中称为服务，具体取决于其端口类型。PCI Express 端口的服务包括本地热插拔支持（HP）、电源管理事件支持（PME）、高级错误报告支持（AER）和虚拟通道支持（VC）。这些服务可以由单个复杂驱动程序处理，也可以由相应的服务驱动程序单独分发和处理。

## 2.3. 为什么使用 PCI Express 端口总线驱动程序

在现有的 Linux 内核中，Linux 设备驱动程序模型只允许单个驱动程序处理物理设备。PCI Express 端口是具有多个不同服务的 PCI-PCI 桥设备。为了维护一个干净简单的解决方案，每个服务可能都有自己的软件服务驱动程序。在这种情况下，几个服务驱动程序将竞争单个 PCI-PCI 桥设备。例如，如果首先加载 PCI Express 根端口本机热插拔服务驱动程序，它会声明一个 PCI-PCI 桥接根端口。因此，内核不会为该根端口加载其他服务驱动程序。换句话说，使用当前驱动程序模型，不可能在 PCI-PCI 桥设备上同时加载和运行多个服务驱动程序。

要使多个服务驱动程序同时运行，需要有一个 PCI Express 端口总线驱动程序，该驱动程序管理所有填充的 PCI Express 端口，并根据需要将所有提供的服务请求分发给相应的服务驱动程序。下面列出了使用 PCI Express 端口总线驱动程序的一些关键优势：

- 允许多个服务驱动程序在 PCI-PCI 桥端口设备上同时运行。
- 允许以独立的分阶段方法实现服务驱动程序。
- 允许一个服务驱动程序在多个 PCI-PCI 桥端口设备上运行。
- 管理 PCI-PCI 桥端口设备的资源并将其分配给请求的服务驱动程序。

## 2.4. 配置 PCI Express 端口总线驱动程序与服务驱动程序

### 2.4.1. 在内核中包含 PCI Express 端口总线驱动程序支持

包含 PCI Express 端口总线驱动程序取决于内核配置中是否包括 PCI Express 支持。当在内核中启用 PCI Express 支持时，内核将自动包含 PCI Express 端口总线驱动程序作为内核驱动程序。

### 2.4.2. 启用服务驱动程序支持

PCI 设备驱动程序是基于 Linux 设备驱动程序模型实现的。所有服务驱动程序都是 PCI 设备驱动程序。如上所述，一旦内核加载了 PCI Express 端口总线驱动程序，就不可能加载任何服务驱动程序。为了满足 PCI Express 端口总线驱动程序模型，需要对现有服务驱动程序进行一些最小的更改，这些更改不会对现有服务驱动程序的功能产生影响。

服务驱动程序需要使用下面显示的两个 API 来向 PCI Express 端口总线驱动程序注册其服务（见第 5.2.1 和 5.2.2 节）。重要的是，在调用这些 API 之前，服务驱动程序要初始化头文件 /include/linux/pcieport_if.h 中包含的 pcie_port_service_driver 数据结构。否则将导致标识不匹配，从而阻止 PCI Express 端口总线驱动程序加载服务驱动程序。

#### 2.4.2.1 pcie_port_service_register

```c
	int pcie_port_service_register(struct pcie_port_service_driver *new)
```

此 API 将取代 Linux 驱动程序模型的 pci_register_driver API。服务驱动程序应该始终在模块 init 处调用 pcie_port_service_register 。请注意，在加载服务驱动程序后，不再需要诸如 pci_enable_device(dev) 和 pci_set_master(dev) 。

#### 2.4.2.2 pcie_port_service_unregister

```c
	void pcie_port_service_unregister(struct pcie_port_service_driver *new)
```

pcie_port_service_unregister 替换 Linux 驱动程序模型的 pci_unregister_driver。当模块退出时，它总是由服务驱动程序调用。

#### 2.4.2.3. 样例代码

下面是初始化端口服务驱动程序数据结构的服务驱动程序示例代码。

```c
    static struct pcie_port_service_id service_id[] = { {
      .vendor = PCI_ANY_ID,
      .device = PCI_ANY_ID,
      .port_type = PCIE_RC_PORT,
      .service_type = PCIE_PORT_SERVICE_AER,
      }, { /* end: all zeroes */ }
    };

    static struct pcie_port_service_driver root_aerdrv = {
      .name               = (char *)device_name,
      .id_table   = &service_id[0],

      .probe              = aerdrv_load,
      .remove             = aerdrv_unload,

      .suspend    = aerdrv_suspend,
      .resume             = aerdrv_resume,
    };
```

以下是用于注册/注销服务驱动程序的示例代码。

```c
    static int __init aerdrv_service_init(void)
    {
      int retval = 0;

      retval = pcie_port_service_register(&root_aerdrv);
      if (!retval) {
        /*
        * FIX ME
        */
      }
      return retval;
    }

    static void __exit aerdrv_service_exit(void)
    {
      pcie_port_service_unregister(&root_aerdrv);
    }

    module_init(aerdrv_service_init);
    module_exit(aerdrv_service_exit);
```

## 2.5. 可能的资源冲突

### 2.5.1. MSI 和 MSI-X 向量资源

一旦在设备上启用 MSI或 MSI-X 中断，它将保持此模式，直到它们再次被禁用。由于同一 PCI-PCI 桥端口的服务驱动程序共享同一物理设备，如果单个服务驱动程序启用或禁用 MSI/MSI-X 模式，则可能导致不可预测的行为。

为了避免这种情况，不允许所有服务驱动程序在其设备上切换中断模式。PCI Express 端口总线驱动程序负责确定中断模式，这对服务驱动程序应该是透明的。服务驱动程序只需要知道分配给结构 pcie_device 的字段 IRQ 的向量 IRQ ，该向量在 PCI Express 端口总线驱动程序探测每个服务驱动程序时传入。服务驱动程序应该使用 (structpcie_device*) dev->irq 来调用 request_irq/free_irq 。此外，中断模式存储在结构 pcie_device 的 interrupt_mode 字段中。

### 2.5.2. PCI 内存/IO 映射区域

PCI Express 电源管理（PME）、高级错误报告（AER）、热插拔（HP）和虚拟通道（VC）的服务驱动程序访问 PCI Express 端口上的 PCI 配置空间。在任何情况下，访问的寄存器都是相互独立的。这个补丁假设所有服务驱动程序都表现良好，不会覆盖其他服务驱动程序的配置设置。

### 2.5.3. PCI 配置寄存器

每个服务驱动程序都在其自己的功能结构体上运行其 PCI 配置操作，但 PCI Express 功能结构体除外，后者在包括服务驱动程序在内的许多驱动程序之间共享。RMW 功能访问器（pcie_Capability_clear_and_set_word()、pcie_cpability_set_word() 和 pcie_cabability_clear_word()）保护选定的 PCI Express 功能寄存器集（链路控制寄存器和根控制寄存器）。对这些寄存器的任何更改都应该使用 RMW 访问器来执行，以避免由于并发更新而导致的问题。有关受保护寄存器的最新列表，请参阅 pcie_capability_clear_and_set_word() 。
