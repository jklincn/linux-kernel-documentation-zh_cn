> 原文：[PCI Peer-to-Peer DMA Support — The Linux Kernel documentation](https://docs.kernel.org/driver-api/pci/p2pdma.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.19
>
> 修订：2024.07.27

# PCI P2P DMA 支持

PCI 总线在执行总线上两个设备之间的 DMA 传输方面有相当不错的支持。这种类型的事务被称为对等（Peer-to-Peer 或 P2P）事务。然而，有许多问题使得以完全安全的方式执行 P2P 事务变得棘手。

其中一个最大的问题是 PCI 并不要求在层次结构域之间转发事务，而在 PCIe 中，每个根端口定义了一个单独的层次结构域。更糟糕的是，没有简单的方法来确定给定的根复杂是否支持这一点。（参见 PCIe r4.0，第 1.3.1 节）。因此，截至撰写本文时，内核仅支持在所有端点都位于同一 PCI 桥后面时执行 P2P，因为此类设备都在同一 PCI 层次结构域中，并且规范保证层次结构内的所有事务都是可路由的，但不要求在层次结构之间进行路由。

第二个问题是，为了利用 Linux 中现有的接口，用于 P2P 事务的内存需要由结构体页面（struct pages）支持。然而，PCI BARs 通常不是缓存一致的，因此这些页面在一些极端情况下会有一些问题，因此开发人员需要对它们的使用非常小心。

## 驱动开发者指南

在一个给定的 P2P 实现中，可能涉及三种或更多不同类型的内核驱动程序：

- 提供者 —— 向其他驱动程序提供或发布 P2P 资源（如内存或门铃寄存器）的驱动程序。
- 客户端 —— 通过设置与资源之间的 DMA 事务来利用资源的驱动程序。。
- 协调者 —— 协调客户端和提供者之间数据流的驱动程序。

在许多情况下，这三种类型之间可能存在重叠（即，一个驱动程序可以同时作为提供者和客户端）。

例如，在 NVMe 目标复制卸载实现中：

- NVMe PCI 驱动程序同时是客户端、提供者和协调者，因为它将任何 CMB（控制器内存缓冲区）公开为 P2P 内存资源（提供者），接受 P2P 内存页作为请求中直接使用的缓冲区（客户端），并且它也可以使用 CMB 作为提交队列条目（协调者）。
- RDMA 驱动程序在这种安排中是一个客户端，因此 RNIC（远程直接存取网卡）可以直接对 NVMe 设备公开的内存进行 DMA 操作。
- NVMe 目标驱动程序（nvmet）可以协调从 RNIC 到 P2P 内存（CMB）再到 NVMe 设备的数据（反之亦然）。

这是当前内核支持的唯一安排，但可以设想对其进行一些微调以实现相同的功能。例如，如果特定 RNIC 添加了一个带有一些内存的 BAR，它的驱动程序可以添加其作为 P2P 提供者的支持，然后 NVMe 目标驱动程序可以在使用的 NVMe 卡不支持 CMB 的情况下使用 RNIC 的内存而不是 CMB。

### 提供者驱动

提供者只需使用 [`pci_p2pdma_add_resource()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_p2pdma_add_resource) 将一个 BAR（或 BAR 的一部分）注册为 P2P DMA 资源。此操作会为所有指定的内存注册页结构体。

之后，它可以选择性地使用 [`pci_p2pmem_publish()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_p2pmem_publish) 将其所有资源发布为 P2P 内存。这将允许任何协调者驱动程序找到并使用这些内存。当以这种方式标记时，资源必须是没有副作用的常规内存。

就目前而言，这种方式相对初级，因为所有资源通常都将是 P2P 内存。未来的工作可能会扩展以包括其他类型的资源，如门铃寄存器。

### 客户端驱动

客户端驱动程序只需像往常一样使用映射 API `dma_map_sg()` 和 `dma_unmap_sg()` 函数，其实现将对具有 P2P 功能的内存做正确的处理。

### 协调者驱动

协调器驱动程序的首要任务是编译一份所有参与给定事务的客户设备的列表。例如，NVMe 目标驱动程序会创建一个包括命名空间块设备和使用中的 RNIC 的列表。如果协调器有权使用特定的 P2P 提供者，它可以使用 `pci_p2pdma_distance()` 检查兼容性；否则，它可以使用 `pci_p2pmem_find()` 找到与所有客户端兼容的内存提供者。如果支持多个提供者，则首先选择最接近所有客户端的提供者。如果有多个提供者距离相同，则随机选择一个（这不是任意选择，而是真正的随机选择）。此函数返回作为提供者的 PCI 设备的引用，因此当不再需要它时应使用 [`pci_dev_put()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_dev_put) 归还。

一旦选择了一个提供者，协调者就可以使用 [`pci_alloc_p2pmem()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_alloc_p2pmem)  和 [`pci_free_p2pmem()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_free_p2pmem) 从提供者那里分配 P2P 内存。[`pci_p2pmem_alloc_sgl()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_p2pmem_alloc_sgl) 和 [`pci_p2pmem_free_sgl()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_p2pmem_free_sgl) 是用于分配带有 P2P 内存的散列表的便捷函数。

> 译者注：散列表（scatter-gather lists，SGL）是一种内核数据结构，用于管理非连续内存块的列表。其中分散（scatter）表示数据被分散存储在内存中的多个不连续区域，而聚集（gather）则表示这些分散的数据被聚集在一起形成连续的逻辑数据流。

### 页结构体注意事项

驱动程序编写者应该非常小心，不要将这些特殊的页结构体传递给未准备好处理它的代码。目前，内核接口没有任何检查来确保这一点。这显然排除了将这些页面传递给用户空间的可能性。

P2P 内存在技术上也属于 IO 内存，但背后不应该有任何的副作用。因此，加载和存储的顺序应该不重要，不需要使用 ioreadX()、iowriteX() 及类似函数。

## P2P DMA 支持库

> 译者注：此处不翻译 API 内容，有关 API 参数、描述、注意、返回值等信息可以查看[原文](https://docs.kernel.org/driver-api/pci/p2pdma.html#p2p-dma-support-library)。

| 函数                                                         | 功能                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| int **pci_p2pdma_add_resource**(struct pci_dev *pdev, int bar, size_t size, u64 offset) | 添加用作 P2P 内存的内存                                      |
| int **pci_p2pdma_distance_many**(struct pci_dev *provider, struct [device](https://docs.kernel.org/driver-api/infrastructure.html#c.device) **clients, int num_clients, bool verbose) | 确定 P2P DMA 提供者与正在使用的客户端之间的累积距离          |
| bool **pci_has_p2pmem**(struct pci_dev *pdev)                | 检查给定的 PCI 设备是否发布了任何 P2P 内存                   |
| struct pci_dev ***pci_p2pmem_find_many**(struct [device](https://docs.kernel.org/driver-api/infrastructure.html#c.device) **clients, int num_clients) | 查找与指定的客户端列表和最短距离兼容的 P2P DMA 内存设备      |
| void ***pci_alloc_p2pmem**(struct pci_dev *pdev, size_t size) | 分配 P2P DMA 内存                                            |
| void **pci_free_p2pmem**(struct pci_dev *pdev, void *addr, size_t size) | 释放 P2P DMA 内存                                            |
| pci_bus_addr_t **pci_p2pmem_virt_to_bus**(struct pci_dev *pdev, void *addr) | 返回使用 [`pci_alloc_p2pmem()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_alloc_p2pmem) 获得的给定虚拟地址的 PCI 总线地址 |
| struct scatterlist ***pci_p2pmem_alloc_sgl**(struct pci_dev *pdev, unsigned int *nents, u32 length) | 分配 P2P DMA 内存并将其存储在散列表中                        |
| void **pci_p2pmem_free_sgl**(struct pci_dev *pdev, struct scatterlist *sgl) | 释放由 [`pci_p2pmem_alloc_sgl()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_p2pmem_alloc_sgl) 分配的散列表 |
| void **pci_p2pmem_publish**(struct pci_dev *pdev, bool publish) | 发布 P2P DMA 内存以供由 pci_p2pmem_find() 找到的其他设备使用 |
| int **pci_p2pdma_enable_store**(const char *page, struct pci_dev **p2p_dev, bool *use_p2pdma) | 解析 configfs/sysfs 属性存储以启用 P2P DMA                   |
| ssize_t **pci_p2pdma_enable_show**(char *page, struct pci_dev *p2p_dev, bool use_p2pdma) | 显示 configfs/sysfs 属性，指示是否启用了 P2P DMA             |
