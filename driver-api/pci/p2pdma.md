> 原文：[PCI Peer-to-Peer DMA Support — The Linux Kernel documentation](https://docs.kernel.org/driver-api/pci/p2pdma.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.19

# PCI P2P DMA 支持

PCI总线对在总线上的两个设备之间执行 DMA 传输有相当不错的支持。在后文中这种类型的事务被称为对等（或 P2P）。然而，有许多问题使得 P2P 事务难以以完全安全的方式进行。

最大的问题之一是 PCI 不需要在层次结构域之间转发事务，而在 PCIe 中，每个根端口都定义了一个单独的层次结构域。更糟糕的是，没有简单的方法来确定给定的根复合体是否支持这一点。（请参阅 PCIe r4.0，第 1.3.1 节）。因此，在撰写本文时，内核仅在所涉及的端点都位于同一个 PCI 桥后面时才支持进行 P2P 操作，因为这样的设备都位于同一个 PCI 层次结构域中，规范保证了层次结构内的所有事务都是可路由的，但它不要求在层次结构之间进行路由。

第二个问题是，要利用 Linux 中现有的接口，用于 P2P 事务的内存需要由页结构体支持。然而，PCI BAR 通常不具有缓存一致性，因此这些页存在一些小问题，因此开发人员需要小心处理它们。

> 译者注：进行 DMA 操作时，需要确保内存是连续的且可访问的。因此，当进行 P2P 事务时，需要确保用于 DMA 的内存是由**页结构体**（struct page，linux 5.4 版本是在 /include/linux/mm_types.h 中定义）支持的，以便 Linux 内核能够正确处理这些页面。

## 驱动开发者指南

在给定的 P2P 实现中，可能存在三种或更多种不同类型的内核驱动程序：

- 提供者 —— 向其他驱动程序提供或发布 P2P 资源（如内存或门铃寄存器）的驱动程序。
- 客户端 —— 通过设置与资源之间的 DMA 事务来利用资源的驱动程序。
- 协调者 —— 协调客户端和提供者之间数据流的驱动程序。

在许多情况下，这三种类型之间可能存在重叠（即，驱动程序通常既是提供者又是客户端）。

例如，在 NVMe 目标复制卸载实现中：

- NVMe PCI 驱动程序既是客户端、提供者又是协调者，因为它将任何 CMB（控制器内存缓冲区）公开为 P2P 内存资源（提供者），它接受 P2P 内存页作为请求中直接使用的缓冲区（客户端），并且还可以将 CMB 用作提交队列条目（协调者）。
- RDMA 驱动程序在这种安排中是客户端，以便 RNIC 可以直接将数据传输到 NVMe 设备公开的内存中。
- NVMe 目标驱动程序（nvmet）可以将数据从 RNIC 协调到 P2P 内存（CMB），然后再传输到 NVMe 设备（反之亦然）。

这是目前内核支持的唯一安排，但可以想象对其进行轻微调整以实现相同的功能。例如，如果一个特定的 RNIC 添加了一个一些带有内存的 BAR，那么它的驱动程序可以添加作为 P2P 提供者的支持，然后 NVMe 目标可以在 NVMe 卡不支持 CMB 的情况下使用 RNIC 的内存。

### 提供者驱动

提供者只需使用 [`pci_p2pdma_add_resource()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_p2pdma_add_resource) 将一个 BAR（或 BAR 的一部分）注册为 P2P DMA 资源。这将为所有指定的内存注册页结构体。

之后，它可以选择性地使用 [`pci_p2pmem_publish()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_p2pmem_publish) 将其所有资源发布为 P2P 内存。这将允许任何协调者驱动程序找到并使用内存。在这种标记方式下，资源必须是没有任何副作用的常规内存。

就目前而言，这是相当基础的，因为所有资源通常都是 P2P 内存。未来的工作可能会将其扩展到包括门铃寄存器等其他类型的资源。

### 客户端驱动

客户端驱动程序只需像往常一样使用映射 API **dma_map_sg()** 和 **dma_unmap_sg()** 函数，它们将为具有 P2P 功能的内存做正确的事情。

### 协调者驱动

协调者驱动程序必须完成的第一项任务是编译一个给定事务中将涉及的所有客户端设备的列表。例如，NVMe 目标驱动程序创建一个列表，其中包括正在使用的名称空间块设备和 RNIC。如果协调者可以访问特定的 P2P 提供者以供使用，它可以使用 `pci_p2pdma_distance()` 来检查兼容性；否则，它可以使用 `pci_p2pmem_find()` 找到与所有客户端兼容的内存提供者。如果支持多个提供者，则首先选择最接近所有客户端的提供者。如果有多个提供者距离相等，则随机选择一个（这不是任意的，而是真正的随机选择）。此函数返回提供者 PCI 设备的引用，因此当不再需要它时应使用 [`pci_dev_put()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_dev_put) 返回。

一旦选择了一个提供者，协调者就可以使用 [`pci_alloc_p2pmem()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_alloc_p2pmem)  和 [`pci_free_p2pmem()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_free_p2pmem) 从提供者分配 P2P 内存。[`pci_p2pmem_alloc_sgl()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_p2pmem_alloc_sgl) 和 [`pci_p2pmem_free_sgl()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_p2pmem_free_sgl) 是用于使用 P2P 内存分配分散-聚集列表的便捷函数。

> 译者注：分散-聚集列表（scatter-gather lists，SGL）是一种内核数据结构，用于管理非连续内存块的列表。其中分散（scatter）表示数据被分散存储在内存中的多个不连续区域，而聚集（gather）则表示这些分散的数据被聚集在一起形成连续的**逻辑**数据流。

### 页结构体注意事项

驱动程序编写者应该非常小心，不要将这些特殊的页结构体传递给没有准备好的代码。目前，内核接口没有任何检查来确保这一点。这显然排除了将这些页面传递给用户空间的可能性。

P2P 内存在技术上也属于 IO 内存，但背后不应该有任何的副作用。因此，加载和存储的顺序不应该重要，并且不应该需要使用 ioreadX()、iowriteX() 等函数。

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
| pci_bus_addr_t **pci_p2pmem_virt_to_bus**(struct pci_dev *pdev, void *addr) | 返回使用 [`pci_alloc_p2pmem()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_alloc_p2pmem) 获得的给定虚拟地址的 PCI总线地址 |
| struct scatterlist ***pci_p2pmem_alloc_sgl**(struct pci_dev *pdev, unsigned int *nents, u32 length) | 分配 P2P DMA 内存并将其存储在分散-聚集列表中                 |
| void **pci_p2pmem_free_sgl**(struct pci_dev *pdev, struct scatterlist *sgl) | 释放由 [`pci_p2pmem_alloc_sgl()`](https://docs.kernel.org/driver-api/pci/p2pdma.html#c.pci_p2pmem_alloc_sgl) 分配的分散-聚集列表 |
| void **pci_p2pmem_publish**(struct pci_dev *pdev, bool publish) | 发布 P2P DMA 内存以供由 pci_p2pmem_find() 找到的其他设备使用 |
| int **pci_p2pdma_enable_store**(const char *page, struct pci_dev **p2p_dev, bool *use_p2pdma) | 解析 configfs/sysfs 属性存储以启用 P2P DMA                   |
| ssize_t **pci_p2pdma_enable_show**(char *page, struct pci_dev *p2p_dev, bool use_p2pdma) | 显示 configfs/sysfs 属性，指示是否启用了 P2P DMA             |

