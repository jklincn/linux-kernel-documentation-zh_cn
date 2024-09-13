> 原文：[1. How To Write Linux PCI Drivers  — The Linux Kernel documentation](https://docs.kernel.org/PCI/pci.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.15
>
> 修订：2024.09.13

# 1. 如何编写 Linux PCI 驱动程序

PCI 的世界是广阔的，充满了（大部分令人不快的）惊喜。由于每个 CPU 体系结构都实现不同的芯片组，并且 PCI 设备有不同的要求（呃，“功能”），因此 Linux 内核中的 PCI 支持并不像人们希望的那样简单。这篇短文试图向所有潜在的驱动程序开发者介绍用于 PCI 设备驱动程序的 Linux API。

更完整的资源是 Jonathan Corbet、Alessandro Rubini 和 Greg Kroah-Hartman 编写的第三版《Linux 设备驱动程序》。LDD3 可以免费获得（在知识共享许可下）： https://lwn.net/Kernel/LDD3/。

但是，请记住，所有文档都会受到“位腐烂”的影响。如果事情没有按照这里描述的那样进行，请参考源代码。

> 译者注：位腐烂原文为“bit rot”，是一个非正式术语，用来形容数据、软件或者数字媒体随时间逐渐降解或损坏的现象。这个术语常用于计算机科学和信息技术中，强调了维护和更新数据或软件重要性，以避免这种逐渐的退化。

请将有关 Linux PCI API 的问题/评论/补丁发送到“Linux PCI” \<linux-pci@atrey.karlin.mff.cuni.cz\> 邮件列表。

## 1.1. PCI 驱动程序的结构

PCI 驱动程序通过 pci_register_driver() “发现”系统中的 PCI 设备。事实上，情况正好相反。当 PCI 通用代码发现新设备时，将通知与“描述”相匹配的驱动程序。详细情况如下。

pci_register_driver() 将大部分设备探测的工作留给 PCI 层，并支持设备的在线插入/移除【从而在单个驱动程序中支持可热插拔的 PCI、CardBus 和 Express-Card】。pci_register_driver() 调用需要传入一个函数指针表，从而指示驱动程序的高层结构。

一旦驱动程序了解 PCI 设备并取得所有权，驱动程序通常需要执行以下初始化：

- 启用设备
- 请求 MMIO/IOP 资源
- 设置 DMA 掩码大小（包括一致性 DMA 和流式 DMA）
- 分配和初始化共享控制数据（pci_allocate_coherent()）
- 访问设备配置空间（如果需要）
- 注册 IRQ 处理程序（[request_irq()](https://docs.kernel.org/core-api/genericirq.html#c.request_irq)）
- 初始化非 PCI（即芯片的 LAN/SCSI/ 等部分）
- 启用 DMA/处理 引擎

当使用完设备后，可能需要卸载模块，驱动程序需要采取以下步骤：

- 禁止设备生成 IRQ
- 释放 IRQ（[free_irq()](https://docs.kernel.org/core-api/genericirq.html#c.free_irq)）
- 停止所有 DMA 活动
- 释放 DMA 缓冲区（包括一致性 DMA 和流式 DMA）
- 从其他子系统（例如 scsi 或 netdev）上注销
- 释放 MMIO/IO端口 资源
- 禁用设备

这些主题中的大部分将在以下部分中介绍。其余的内容请参考 LDD3 或 <linux/pci.h> 。

如果没有配置 PCI 子系统（没有设置 CONFIG_PCI ），下面描述的大多数 PCI 函数被定义为内联函数，要么完全为空，要么只是返回一个适当的错误代码，以避免在驱动程序中出现大量的 ifdef 。

## 1.2. pci_register_driver() 调用

PCI 设备驱动程序在初始化过程中调用 `pci_register_driver()` ，并提供一个指向描述驱动程序的结构体的指针（ `struct pci_driver` ）：

```c
// PCI驱动程序结构体
struct pci_driver {
    const char              *name;
    const struct pci_device_id *id_table;
    int (*probe)(struct pci_dev *dev, const struct pci_device_id *id);
    void (*remove)(struct pci_dev *dev);
    int (*suspend)(struct pci_dev *dev, pm_message_t state);
    int (*resume)(struct pci_dev *dev);
    void (*shutdown)(struct pci_dev *dev);
    int (*sriov_configure)(struct pci_dev *dev, int num_vfs);
    int (*sriov_set_msix_vec_count)(struct pci_dev *vf, int msix_vec_count);
    u32 (*sriov_get_vf_total_msix)(struct pci_dev *pf);
    const struct pci_error_handlers *err_handler;
    const struct attribute_group **groups;
    const struct attribute_group **dev_groups;
    struct device_driver    driver;
    struct pci_dynids       dynids;
    bool driver_managed_dma;
};
```

- `name`：驱动程序名称。
- `id_table`：指向驱动程序感兴趣的设备 ID 表的指针。大多数驱动程序应该使用宏 MODULE_device_table（pci, …）导出此表。
- `probe`：对于与 ID 表匹配且尚未被其他驱动程序“拥有”的所有 PCI 设备，将调用此探测函数（在为现有设备执行 pci_register_driver() 期间，或在插入新设备后执行）。此函数为 ID 表中条目与设备匹配的每个设备传递一个 “struct pci_dev*” 。当驱动程序选择获得设备的“所有权”或错误代码（负数）时，探测函数返回零。探测函数总是从进程上下文中调用，因此它可以休眠。
- `remove`：每当删除此驱动程序处理的设备时（在注销驱动程序期间或从可热插拔插槽中手动拔出时），就会调用 remove() 函数。remove 函数总是从进程上下文中调用，因此它可以休眠。
- `suspend`：将设备置于低功率状态。
- `resume`：将设备从低功率状态唤醒。（有关PCI电源管理和相关功能的说明，请参阅[PCI电源管理](https://docs.kernel.org/power/pci.html)。）
- `shutdown`：挂入 reboot_notifier_list (kernel/sys.c) 。旨在停止任何空闲 DMA 操作。用于在重新启动之前启用局域网唤醒（NIC）或更改设备的电源状态。例如 drivers/net/e100.c。
- `sriov_configure`：可选的驱动程序回调，允许通过 sysfs 的 “sriov_numvfs”文件配置要启用的 VF 数量。
- `sriov_set_msix_vec_count`：PF 驱动程序回调以更改 VF 上 MSI-X 矢量的数量。通过 sysfs 的 “sriov_vf_mix_count” 触发。这将更改 VF 消息控制寄存器中的 MSI-X 表大小。
- `sriov_get_vf_total_msix`：PF 驱动程序回调，以获得可用于分配给 VF 的 MSI-X 向量的总数。
- `err_handler`：见 [PCI 错误恢复](https://docs.kernel.org/PCI/pci-error-recovery.html)。
- `groups`：Sysfs 属性组。
- `dev_groups`：绑定到驱动程序后将创建的附加到设备的属性。
- `driver`：驱动程序模型结构
- `dynids`：动态添加的设备 ID 的列表。
- `driver_managed_dma`：设备驱动程序不使用内核 DMA API 进行 DMA 。对于大多数设备驱动程序，只要所有 DMA 都是通过内核 DMA API 处理的，就不需要关心这个标志。对于一些特殊的驱动程序，例如 VFIO 驱动程序，他们知道如何自己管理 DMA 并设置此标志，以便 IOMMU 层允许他们设置和管理自己的 I/O 地址空间。

id_table 是一个 pci_device_id 数组，以全零的条目结尾。通常情况下，使用 `static const` 进行定义会更好。

```c
// PCI设备ID结构体
struct pci_device_id {
    __u32 vendor, device;
    __u32 subvendor, subdevice;
    __u32 class, class_mask;
    kernel_ulong_t driver_data;
    __u32 override_only;
};
```

- `vendor`：要匹配的供应商 ID（或 PCI_ANY_ID）

- `device`：要匹配的设备 ID（或 PCI_ANY_ID）

- `subvendor`：要匹配的子系统供应商 ID（或 PCI_ANY_ID）

- `subdevice`：要匹配的子系统设备 ID（或 PCI_ANY_ID）

- `class`：要匹配的设备类、子类和“接口”。有关类的完整列表，请参阅 PCI 本地总线规范的附录 D 或 include/linux/PCI_ids.h。大多数驱动程序不需要指定 class/class_mask，因为通常情况下，有 vendor/device 字段就足够了。

- `class_mask`：在比较 class 字段的子字段时的限制条件。有关用法示例，请参阅 drivers/scsi/sym53c8xx_2/。

  > 译者注：简而言之，class_mask 决定了在 class 字段中哪些子字段会被用于比较。

- `driver_data`：驱动程序专用的数据。大多数驱动程序不需要使用 driver_data 字段。最佳实践是使用 driver_data 作为等效设备类型的静态列表的索引，而不是将其用作指针。

- `override_only`：仅当 dev->driver_override 是此驱动程序时才匹配。

大多数驱动程序只需要 PCI_DEVICE() 或 PCI_DEVICE_CLASS() 来设置 pci_device_id 表。

新的 PCI ID 可以在运行时添加到设备驱动程序 PCI_IDs 表中，如下所示：

```bash
echo "vendor device subvendor subdevice class class_mask driver_data" > \
/sys/bus/pci/drivers/{driver}/new_id
```

所有字段都以十六进制值的形式传入（没有前导 0x）。vendor 和 device 是必填字段，其他字段是可选的。用户只需根据需要传递必要的可选字段：

- subvendor 和 subdevice 字段默认为 PCI_ANY_ID（FFFFFFFF）
- class 和 class_mask 字段默认为 0
- driver_data 默认为 0UL
- override_only 字段默认为 0

请注意，driver_data 必须与驱动程序中定义的任何 pci_device_id 条目所使用的值相匹配。这使得如果所有 pci_device_id 条目都具有非零的 driver_data 值，则 driver_data 字段变为必需的。

一旦添加，对于其 pci_ids 列表中列出的任何未声明的 PCI 设备，都会调用驱动程序的探测例程。

当驱动程序退出时，它只需调用 pci_unregister_driver()，PCI 层会自动为驱动程序处理的所有设备调用移除钩子函数。

### 1.2.1. 驱动程序函数/数据的“属性”

请在适当的地方标记初始化和清理函数（相应的宏在 <linux/init.h> 中定义）：

|        |                                      |
| ------ | ------------------------------------ |
| __init | 初始化代码。在驱动程序初始化后丢弃。 |
| __exit | 退出代码。对于非模块化驱动程序忽略。 |

关于何时/何地使用上述属性的提示：

- [module_init()](https://docs.kernel.org/driver-api/basics.html#c.module_init) / [module_exit()](https://docs.kernel.org/driver-api/basics.html#c.module_exit) 函数（以及从中调用 \_only\_ 的所有初始化函数）应标记为 \_\_init/\_\_exit 。
- 不要标记 [struct pci_driver](https://docs.kernel.org/PCI/pci.html#c.pci_driver)
- 如果不确定要使用哪个标记时，请不要标记函数。不标记比标记错要好。

## 1.3. 如何手动查找 PCI 设备

PCI 驱动程序应该有充分的理由不使用 pci_register_driver() 接口来搜索 PCI 设备。PCI 设备由多个驱动程序控制的主要原因是一个 PCI 设备实现了多种不同的硬件服务。例如，集成了串口/并口/软盘控制器的设备。

可以使用以下构造进行手动搜索：

通过供应商和设备 ID 搜索：

```c
    struct pci_dev *dev = NULL;
    while (dev = pci_get_device(VENDOR_ID, DEVICE_ID, dev))
            configure_device(dev);
```

通过类 ID 搜索（以类似方式迭代）：

```c
    pci_get_class(CLASS_ID, dev)
```

通过供应商/设备和子系统供应商/设备 ID 进行搜索：

```c
	pci_get_subsys(VENDOR_ID,DEVICE_ID, SUBSYS_VENDOR_ID, SUBSYS_DEVICE_ID, dev)
```

您可以使用常量 PCI_ANY_ID 作为 VENDOR_ID 或 DEVICE_ID 的通配符替换。例如，这允许搜索特定供应商的任何设备。

这些函数是热插拔安全的。它们会增加返回的 pci_dev 上的引用计数。您最终必须（可能在模块卸载时）通过调用 [pci_dev_put()](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_dev_put) 来减少这些设备上的引用计数。

## 1.4. 设备初始化步骤

如引言中所述，大多数 PCI 驱动程序需要以下步骤进行设备初始化：

- 启用设备
- 请求 MMIO/IOP 资源
- 设置 DMA 掩码大小（包括一致性 DMA 和流式 DMA）
- 分配和初始化共享控制数据（pci_allocate_coherent()）
- 访问设备配置空间（如果需要）
- 注册 IRQ 处理程序（[request_irq()](https://docs.kernel.org/core-api/genericirq.html#c.request_irq)）
- 初始化非 PCI（即芯片的 LAN/SCSI/ 等部分）
- 启用 DMA/处理 引擎

驱动程序可在任何时候访问 PCI 配置空间寄存器。（好吧，几乎是这样的。当运行 BIST 时，配置空间可能会消失……但这只会导致 PCI 总线主机中止，并且配置读取将返回无效数据）。

> 译者注：BIST 指 Built-In Self-Test （嵌入式自我测试）。这是一种硬件功能，允许设备检查自身的主要功能和部件是否正常工作。

### 1.4.1. 启用 PCI 设备

在接触任何设备寄存器之前，驱动程序需要通过调用 pci_enable_device() 来启用 PCI 设备。这将：

- 唤醒设备，如果设备处于暂停状态，
- 为设备分配 I/O 和内存区域（如果 BIOS 没有做的话），
- 分配一个 IRQ （如果 BIOS 没有做的话）。

> [!NOTE]
>
> [`pci_enable_device()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_enable_device) 可能会失败！请检查返回值。

> [!WARNING]
>
> 操作系统 BUG：在启用这些资源之前，我们不会检查资源分配。如果我们在调用 pci_enable_device () 之前调用 pci_request_resources()，顺序会更有意义。目前，当两个设备被分配到相同的范围时，设备驱动程序无法检测到错误。这不是一个常见的问题，也不太可能很快得到解决。
>
> 这个问题之前已经讨论过，但截至 2.6.19 版本还未改变：https://lore.kernel.org/r/20060302180025.GC28895@flint.arm.linux.org.uk/

[`pci_set_master()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_set_master) 将通过在 PCI_COMMAND 寄存器中设置总线主位（bus master bit）来启用 DMA 。它还修复了延迟定时器值，如果它被 BIOS 设置为伪造的值。[`pci_clear_master()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_clear_master) 将通过清除总线主位来禁用 DMA 。

如果 PCI 设备可以使用 PCI Memory-Write-Invalidate 事务，请调用 [`pci_set_mwi()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_set_mwi) 。这将启用 Mem-Wr-Inval 的 PCI_COMMAND 位，并确保正确设置高速缓存行大小寄存器。请检查 [`pci_set_mwi()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_set_mwi) 的返回值，因为并非所有体系结构或芯片组都支持 Memory-Write-Invalidate 。或者，如果 Mem-Wr-Inval 不是必需的但使用它会更好，请调用 [`pci_try_set_mwi()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_try_set_mwi)，让系统尽最大努力启用 Mem-Wr-Inval 。

> 译者注：Memory-Write-Invalidate 是一种 PCI 总线事务类型，可以优化内存写入操作。它允许写入并且完全覆盖一个或多个内存块的内容，而无需像平常的内存写入一样要先读取整个内存块，修改相应的部分，然后把整个块写回到内存。使用 Memory-Write-Invalidate 事务可以减少了总线上的数据流量，提高了写入操作的效率，特别是要写入大量数据时。不过它要求硬件和驱动程序支持这一特性，以及操作系统能够正确地管理这种类型的事务。

### 1.4.2. 请求 MMIO/IOP 资源

内存（MMIO）和 I/O 端口地址不应该直接从 PCI 设备配置空间读取。应该使用 pci_dev 结构体中的值，因为 PCI “总线地址” 可能被体系结构/芯片组特定的内核支持重新映射到“主机物理”地址。

有关如何访问设备寄存器或设备内存的信息，请参阅 [io_mapping 函数](https://docs.kernel.org/driver-api/io-mapping.html)。

设备驱动程序需要调用 [`pci_request_region()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_request_region) 来验证没有其他设备正在使用相同的地址资源。相反，驱动程序应该在调用 [`pci_disable_device()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_disable_device) 之后再调用 [`pci_release_region()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_release_region) 。这个想法是为了防止两个设备在同一地址范围内发生冲突。

> [!NOTE]
>
> 请参阅上面的操作系统 BUG 评论。目前（2.6.19），驱动程序只能在调用 [`pci_enable_device()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_enable_device) 后确定 MMIO 和 IO 端口资源的可用性。

[`pci_request_region()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_request_region) 的通用版本是 request_mem_region()（用于 MMIO 范围）和 request_region()（用于 IO 端口范围）。对于那些不被“常规” PCI BARs 描述的地址资源，请使用这些函数。

另请参阅下面的 [`pci_request_selected_regions()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_request_selected_regions) 。

### 1.4.3. 设置 DMA 掩码大小

> [!NOTE]
>
> 如果以下内容没有意义，请参阅[使用通用设备的动态 DMA 映射](../core-api/dma-api.md)。本节只是提醒您，驱动程序需要指示设备的 DMA 功能，而不是 DMA 接口的权威来源。

虽然所有驱动程序都应明确指出 PCI 总线主控器的 DMA 功能（例如 32 位或 64 位），但对流数据具有 32 位以上总线主控能力的设备需要驱动程序通过使用适当的参数调用 DMA_set_mask() 来“注册”此能力。一般来说，这允许在内存物理地址大于 4G 的系统上进行更高效的 DMA 。

所有 PCI-X 和 PCIe 兼容设备的驱动程序必须调用 dma_set_mask() ，因为它们是 64 位 DMA 设备。

同样，如果设备能够直接寻址系统内存中位于 4G 物理地址以上的“一致性内存”，则驱动程序也必须通过调用 dma_set_coherent_mask() 来“注册”此能力。再次强调，这包括所有 PCI-X 和 PCIe 兼容设备的驱动程序。许多 64位 “PCI” 设备（在 PCI-X 之前）和一些 PCI-X 设备能够处理 64 位 DMA 的负载（“流式”）数据，但不适用于控制（“一致性”）数据。

### 1.4.4. 设置共享控制数据

一旦设置了 DMA 掩码，驱动程序就可以分配“一致”（也称为共享）内存。有关 DMA API 的完整描述，请参阅[使用通用设备的动态 DMA 映射](../core-api/dma-api.md)。本节只是提醒您，在设备上启用 DMA 之前需要进行此操作。

### 1.4.5. 初始化设备寄存器

一些驱动程序需要某些寄存器（编程的特定“能力”字段或其他“供应商指定”的）初始化或重置。例如，清除挂起的中断。

### 1.4.6. 注册 IRQ 处理程序

虽然调用 [`request_irq()`](https://docs.kernel.org/core-api/genericirq.html#c.request_irq) 是这里描述的最后一步，但这通常只是初始化设备的另一个中间步骤。这一步骤通常可以推迟到设备打开以供使用时。

IRQ 线路的所有中断处理程序都应向 IRQF_SHARED 注册，并使用 devid 将 IRQ 映射到设备（请记住所有 PCI IRQ 线路都可以共享）。

[`request_irq()`](https://docs.kernel.org/core-api/genericirq.html#c.request_irq) 将中断处理程序和设备句柄与中断号相关联。历史上的中断号代表从 PCI 设备到中断控制器的 IRQ 线路。对于 MSI 和 MSI-X（更多信息见下文），中断号是一个 CPU “向量”。

[`request_irq()`](https://docs.kernel.org/core-api/genericirq.html#c.request_irq) 还启用了中断。在注册中断处理程序之前，请确保设备处于静止状态，并且没有任何挂起的中断。

MSI 和 MSI-X 是 PCI 功能。两者都是“消息信号中断”，通过 DMA 向本地 APIC 写入把中断传递到 CPU 。MSI 和 MSI-X 之间的根本区别在于如何分配多个“向量”。MSI 需要连续的向量块，而 MSI-X 可以分配几个独立的向量块。

可以通过在调用 request_irq() 之前使用 PCI_IRQ_MSI 或 PCI_IRQ_MSIX 标志调用 [`pci_alloc_irq_vectors()`](https://docs.kernel.org/PCI/msi-howto.html#c.pci_alloc_irq_vectors) 来启用 MSI 功能。这将导致 PCI 支持将 CPU 向量数据编程到 PCI 设备功能寄存器中。许多架构、芯片组或 BIOS 不支持 MSI 或 MSI-X，因此使用 PCI_IRQ_MSI 和 PCI_IRQ_MSIX 标志来调用 pci_alloc_irq_vector 可能会失败，因此尝试始终指定 PCI_IRQ_INTX 也很重要。

在调用 pci_alloc_irq_vectors 后，对 MSI/MSI-X 和传统 INTx 具有不同中断处理程序的驱动程序应根据 pci_dev 结构体中的 msi_enabled 和 msix_enabled 标志来选择正确的中断处理程序。

使用 MSI 的（至少）两个非常好的理由：

1. MSI 根据定义是一种独占的中断向量。这意味着中断处理程序不必验证其设备导致了中断。

   > 译者注：在传统的中断机制中，一个中断线路可能被多个设备共享，因此当中断发生时，中断处理程序需要检查是哪个设备触发了中断。然而，使用 MSI 时，每个中断都有一个唯一的向量（即独占的标识），因此当接收到中断时，系统可以直接知道是哪个设备触发了中断，无需进一步检查。这简化了中断处理流程，提高了效率。

2. MSI 避免了 DMA/IRQ 竞争条件。当 MSI 传递时，可以保证 DMA 操作对主机内存的写入对 CPU 是可见的。这对于数据一致性和避免过时的控制数据很重要。这种保证允许驱动程序省略 MMIO 读取以刷新 DMA 流。

   > 译者注：在正常情况下，驱动程序可能需要执行额外的 MMIO 读取操作来确保所有通过 DMA 传输的数据已经完全写入并且对 CPU 可见，这是一种刷新 DMA 流的方法。但是，由于 MSI 机制确保了 DMA 操作的结果在 MSI 到达 CPU 时已经是可见的，因此可以省略这些额外的 MMIO 读取操作，从而简化了驱动程序的实现并提高了效率。

有关 MSI/MSI-X 使用的示例，请参阅 drivers/infiniband/hw/mthca/ 或 drivers/net/tg3.c 。

## 1.5. PCI 设备关闭

卸载 PCI 设备驱动程序时，需要执行以下大部分步骤：

- 禁止设备生成 IRQ
- 释放 IRQ（[free_irq()](https://docs.kernel.org/core-api/genericirq.html#c.free_irq)）
- 停止所有 DMA 活动
- 释放 DMA 缓冲区（包括流式 DMA 和一致性 DMA）
- 从其他子系统（例如 scsi 或 netdev）上注销
- 禁止设备响应 MMIO/IO 端口地址
- 释放 MMIO/IO端口 资源

### 1.5.1. 在设备上停止 IRQ

如何做到这一点是特定于芯片/设备的。如果不执行此操作，如果（并且仅当）IRQ 与另一设备共享时，就有可能出现 “Screaming interrupt” 。

> 译者注：“Screaming interrupt” 用来形容一种特定的硬件或软件问题，其中一个中断源不断地生成中断请求，导致中断处理程序反复被调用，而实际上并没有有效的或新的数据需要处理。这种情况通常发生在设备故障、驱动程序错误或系统配置不当的情况下。由于中断请求不断发生，CPU 将花费大量时间处理这些无效的中断，从而影响系统的正常运行和性能。在共享 IRQ 线路的情况下，如果一个设备错误地持续请求中断，甚至可能影响到共享同一 IRQ 线路的其他设备的正常中断处理。

当共享的 IRQ 处理程序被“解钩”时，使用相同 IRQ 线路的其余设备仍然需要 IRQ 启用。因此，如果被“解钩”的设备断言 IRQ 线路，系统将响应并假设是其余设备断言了 IRQ 线路。由于其他设备不会处理 IRQ，因此系统将“挂起”直到它决定（译者注：原文是“decides”，翻译成“知道”可能会更好） IRQ 不会被处理并屏蔽 IRQ（100,000次迭代后）。一旦共享的 IRQ 被屏蔽，剩余的设备将停止正常工作。这不是一个好情况。

这是另一个使用 MSI 或 MSI-X 的理由，如果 MSI 或 MSI-X 可用。MSI 和 MSI-X 被定义为独占中断，因此不容易受到 “Screaming interrupt” 问题的影响。

### 1.5.2. 释放 IRQ

一旦设备静默（不再有 IRQ），就可以调用 [`free_irq()`](https://docs.kernel.org/core-api/genericirq.html#c.free_irq) 。此函数将在处理任何挂起的 IRQ 后返回控制权，“解钩”驱动程序的 IRQ 处理程序，并最终释放 IRQ（如果没有其他人使用它）。

### 1.5.3. 停止所有 DMA 活动

在尝试释放 DMA 控制数据之前，停止所有 DMA 操作是非常重要的。不这样做可能会导致内存损坏，挂起，以及在某些芯片组上硬崩溃。

在停止 IRQ 之后再停止 DMA 可以避免由于 IRQ 处理程序可能会重新启动 DMA 引擎导致的竞争。

虽然这一步听起来显而易见且微不足道，但一些“成熟”的驱动程序过去并没有正确地完成这一步。

### 1.5.4. 释放 DMA 缓冲区

一旦停止了 DMA，请先清理流式 DMA 。即取消映射数据缓冲区，并将缓冲区返回给“上游”所有者（如果有的话）。

然后清理包含控制数据的“一致性”缓冲区。

有关取消映射接口的详细信息，请参阅[使用通用设备的动态DMA映射](https://docs.kernel.org/core-api/dma-api.html)。

## 1.6. 如何访问 PCI 配置空间

您可以使用 *pci\_(read/write)\_config\_(byte/word/dword)* 来访问由结构体 *pci_dev\** 表示的设备的配置空间。所有这些函数在成功时返回 0，或者返回一个错误代码（*PCIBIOS*_…），该错误代码可以通过 pcibios_strerror 转换为文本字符串。大多数驱动程序都希望访问有效的 PCI 设备不会失败。

如果您没有可用的 pci_dev 结构体，您可以调用 *pci\_bus\_(read/write)\_config\_(byte/word/dword)* 来访问该总线上的给定设备和功能。

如果您需要访问配置头标准部分中的字段，请使用在 <linux/pci.h> 中声明的位置和位的符号名称。

如果您需要访问 PCI 的扩展功能寄存器，只需为特定功能调用 [`pci_find_capability()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_find_capability)，它将为您找到相应的寄存器块。

## 1.7. 其他有趣的函数

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`pci_get_domain_bus_and_slot()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_get_domain_bus_and_slot) | 查找与给定域、总线、插槽和编号相对应的 pci_dev 。如果找到该设备，则会增加其引用计数。 |
| [`pci_set_power_state()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_set_power_state) | 设置 PCI 电源管理状态（0=D0 … 3=D3）                         |
| [`pci_find_capability()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_find_capability) | 在设备的功能列表中查找指定的功能                             |
| pci_resource_start()                                         | 返回给定 PCI 区域的总线起始地址                              |
| pci_resource_end()                                           | 返回给定 PCI 区域的总线结束地址                              |
| pci_resource_len()                                           | 返回 PCI 区域的字节长度                                      |
| pci_set_drvdata()                                            | 为 pci_dev 设置专用驱动程序数据指针                          |
| pci_get_drvdata()                                            | 返回 pci_dev 的专用驱动程序数据指针                          |
| [`pci_set_mwi()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_set_mwi) | 启用 Memory-Write-Invalidate 事务                            |
| [`pci_clear_mwi()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_clear_mwi) | 禁用 Memory-Write-Invalidate 事务                            |

## 1.8. 杂项提示

当向用户显示 PCI 设备名称时（例如，当驱动程序想要告诉用户它发现了什么卡时），请使用 pci_name(pci_dev) 。

始终通过指向 pci_dev 结构体的指针来引用 PCI 设备。所有 PCI 层函数都使用这种标识，这是唯一合理的方式。除了非常特殊的目的外，不要使用总线/插槽/功能号——在具有多个主总线的系统上，它们的语义可能非常复杂。

不要尝试在您的驱动程序中开启“快速背靠背写入”。总线上的所有设备都需要能够做到这一点，因此这是需要由平台和通用代码来处理的事情，而不是单个驱动程序。

> 译者注：快速背靠背写入（Fast Back to Back writes）是一种 PCI 总线通信优化技术，允许数据在没有额外延迟的情况下连续快速地写入到总线上的设备。这种技术主要用于提高数据传输效率和减少写入操作的总线占用时间。然而，要启用 Fast Back to Back writes，总线上的所有设备都必须支持这一技术，并且总线控制器和芯片组也必须能够正确处理这种连续的快速写入操作。这是因为如果总线上的某些设备无法处理这种快速连续写入，可能会导致数据损坏或通信错误。

## 1.9. 供应商和设备ID

除非在多个驱动程序之间共享，否则不要在 include/linux/pci_ids.h 中添加新的设备或供应商 ID 。如果这些有帮助，您可以在驱动程序中添加私有定义，或者只使用纯十六进制常量。

设备 ID 是任意的十六进制数字（由供应商控制），通常只在一个位置使用，即 pci_device_id 表中。

请将新的供应商/设备 ID 提交到 https://pci-ids.ucw.cz/ 。在 https://github.com/pciutils/pciids 有一个 pci.ids 文件的镜像。

## 1.10. 过时的功能

当您尝试将旧的驱动程序移植到新的 PCI 接口时，可能会遇到几个功能。它们不再存在于内核中，因为它们与热插拔或 PCI 域不兼容，或者具有正常锁定。

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| pci_find_device()                                            | 由 [`pci_get_device()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_get_device) 替代 |
| pci_find_subsys()                                            | 由 [`pci_get_subsys()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_get_subsys) 替代 |
| pci_find_slot()                                              | 由 [`pci_get_domain_bus_and_slot()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_get_domain_bus_and_slot) 替代 |
| [`pci_get_slot()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_get_slot) | 由 [`pci_get_domain_bus_and_slot()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_get_domain_bus_and_slot) 替代 |

另一种选择是遍历 PCI 设备列表的传统 PCI 设备驱动程序。这仍然是可能的，但令人气馁。

## 1.11. MMIO 空间和写入过账

将驱动程序从使用 I/O 端口空间转换为使用 MMIO 空间通常需要一些额外的更改。具体来说，需要处理“写入过账”。许多驱动程序（例如 tg3、acenic、sym53c8xx_2）已经做到了这一段。I/O 端口空间保证写入事务在 CPU 继续之前到达 PCI 设备。而写入 MMIO 空间允许 CPU 在事务到达 PCI 设备之前继续。硬件专家将这称为“写入过账”，因为写入完成是在事务到达目的地之前“过账”给CPU的。

因此，对于时间敏感的代码，应该在 CPU 需要等待之前进行其他工作时添加 readl()。经典的 “bit banging” 序列对 I/O 端口空间很有效：

```c
    for (i = 8; --i; val >>= 1) {
            outb(val & 1, ioport_reg);      /* write bit */
            udelay(10);
    }
```

对于 MMIO 空间，相同的序列应该是：

```c
    for (i = 8; --i; val >>= 1) {
            writeb(val & 1, mmio_reg);      /* write bit */
            readb(safe_mmio_reg);           /* flush posted write */
            udelay(10);
    }
```

> 译者注：Bit banging 是一种软件实现的技术，用于生成或解析没有硬件支持的串行通信协议。在这种方法中，软件直接控制硬件的I/O引脚，通过精确地控制时序来模拟串行通信协议的发送和接收。这种技术通常用于简单的通信协议，特别是在没有专用串行通信硬件支持的情况下，或者当需要实现非标准的通信协议时。

重要的是，“safe_mio_reg” 不存在任何干扰设备正确操作的副作用。

另一种需要注意的情况是重置 PCI 设备时。使用 PCI 配置空间读取来刷新 writel()  。如果 PCI 设备预期不会响应 readl()，这将在所有平台上优雅地处理 PCI 主控中止。大多数 x86 平台将允许 MMIO 读取导致主控中止（也称为“软故障”）并返回无效数据（例如 ~0）。但许多 RISC 平台会崩溃（又称“硬故障”）。
