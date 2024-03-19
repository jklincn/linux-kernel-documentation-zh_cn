> 原文：[2. The MSI Driver Guide HOWTO  — The Linux Kernel documentation](https://docs.kernel.org/PCI/msi-howto.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.19
>
> 修订：

# 4. MSI 驱动程序指南

## 4.1. 关于本指南

本指南介绍了消息信号中断（MSI）的基本知识，使用 MSI 相对于传统中断机制的优势，如何将驱动程序更改为使用MSI 或 MSI-X，以及在设备不支持 MSI 时尝试的一些基本诊断。

## 4.2. 什么是 MSI

消息信号中断是指一个来自设备到一个特殊地址的写操作，从而导致 CPU 接收到一个中断。

MSI 功能最初在 PCI 2.2 中指定，后来在 PCI 3.0 中增强，允许单独屏蔽每个中断。MSI-X 功能也与 PCI 3.0 一起引入。它支持在每个设备上有比 MSI 更多的中断，并允许独立配置中断。

设备可以同时支持 MSI 和 MSI-X，但一次只能启用一个。

## 4.3. 为什么使用 MSI

使用 MSI 相比传统的基于引脚的中断有三个优势。

基于引脚的 PCI 中断通常在几个设备之间共享。为了支持这一点，内核必须调用与中断相关联的每个中断处理程序，这会导致整个系统的性能降低。MSI 从不共享，因此不会出现此问题。

当设备将数据写入内存然后引发基于引脚的中断时，有可能中断到达的时间早于所有数据都到达内存的时间（在 PCI-PCI 桥后的设备中这种可能性更大）。为了确保所有数据都已经到达内存，中断处理程序必须读取引发中断的设备上的寄存器。PCI 事务排序规则要求所有数据在寄存器返回值之前到达内存。使用 MSI 可以避免这个问题，因为产生中断的写操作不可能超过数据写操作，所以当中断被触发时，驱动程序知道所有数据都已经到达内存。

> 译者注：即 MSI 的写操作不可能比数据的写操作更早到达，因此触发中断时能确保所以数据已经到达内存。

PCI 设备每个功能只能支持一个基于引脚的中断。驱动程序经常需要查询设备以找出发生了什么事件，这会减慢常见情况下的中断处理速度。使用 MSI，一个设备可以支持更多中断，允许每个中断专门用于不同的目的。一种可能的设计是给不频繁的条件（如错误）设置专属的中断，这允许驱动程序更有效地处理正常的中断处理路径。其他可能的设计包括在网卡中为每个数据包队列或存储控制器中为每个端口分配一个中断。

## 4.4. 如何使用 MSI

PCI 设备被初始化为使用基于引脚的中断。设备驱动程序必须将设备设置为使用 MSI 或 MSI-X 。并非所有机器都正确支持 MSI，对于这些机器，下面描述的 API 将失败，设备将继续使用基于引脚的中断。

### 4.4.1. 在内核中包含 MSI 支持

要支持 MSI 或 MSI-X，内核必须在启用 CONFIG_PCI_MSI 选项的情况下构建。此选项仅在某些体系结构上可用，并且可能取决于正在设置的其他一些选项。例如，在 x86 上，还必须启用 x86_UP_APIC 或 SMP 才能看到 CONFIG_PCI_MSI 选项。

### 4.4.2. 使用 MSI

大部分艰难的工作都是为了 PCI 层中的驱动程序完成的。驱动程序只需请求 PCI 层为该设备设置 MSI 功能。

要自动使用 MSI 或 MSI-X 中断向量，请使用以下功能：

```c
int pci_alloc_irq_vectors(struct pci_dev *dev, unsigned int min_vecs,
              unsigned int max_vecs, unsigned int flags);
```

它为 PCI 设备分配高达 max_ vecs 的中断向量。它返回已分配的向量数量或一个代表错误的负数。如果设备对最小向量数量有要求，则驱动程序可以传递 min_vecs 参数来设置这个限制，如果不能满足最小向量数量，则 PCI 核心将返回 -ENOPSC 。

flags 参数用于指定设备和驱动程序可以使用哪种类型的中断（PCI_IRQ_LEGACY、PCI_IRQ_MSI 和 PCI_IRQ-MSIX）。还提供了一个方便的简写（PCI_IRQ_ALL_TYPES），用于请求任何可能的中断类型。如果设置了 PCI_IRQ_AFFINITY 标志，则 [`pci_alloc_irq_vectors()`](https://docs.kernel.org/PCI/msi-howto.html#c.pci_alloc_irq_vectors) 将会把中断分散到可用的 CPU 上。

要获取传递给 [`request_irq()`](https://docs.kernel.org/core-api/genericirq.html#c.request_irq) 和 [`free_irq()`](https://docs.kernel.org/core-api/genericirq.html#c.free_irq) 的 Linux IRQ 编号以及向量，请使用以下函数：

```c
int pci_irq_vector(struct pci_dev *dev, unsigned int nr);
```

在移除设备之前，应使用以下函数释放所有已分配的资源：

```c
void pci_free_irq_vectors(struct pci_dev *dev);
```

如果设备同时支持 MSI-X 和 MSI 功能，则此 API 将优先使用 MSI-X 功能，而不是 MSI 功能。MSI-X 支持 1 到 2048 之间的任意数量的中断。相反，MSI 被限制为最多 32 个中断（并且必须是 2 的幂）。此外，MSI 中断向量必须连续分配，因此系统可能无法为 MSI 分配与 MSI-X 一样多的向量。在某些平台上，MSI 中断必须全部针对同一组 CPU，而 MSI-X 中断可以全部针对不同的 CPU 。

如果一个设备既不支持 MSI-X 也不支持 MSI，那么它将回退到单个传统 IRQ 向量。

MSI或MSI-X中断的典型用法是尽可能分配更多的向量，可能会达到设备支持的限制。如果 nvec 大于设备支持的数量，它将自动被限制在支持的上限，因此无需事先查询支持的向量数量：

```c
nvec = pci_alloc_irq_vectors(pdev, 1, nvec, PCI_IRQ_ALL_TYPES)
if (nvec < 0)
        goto out_err;
```

如果驱动程序无法或不愿意处理可变数量的 MSI 中断，它可以通过将特定数量的中断作为 “min_vecs” 和 “max_vecs” 参数传递给 [`pci_alloc_irq_vectors()`](https://docs.kernel.org/PCI/msi-howto.html#c.pci_alloc_irq_vectors) 函数来请求特定数量的中断：

```c
ret = pci_alloc_irq_vectors(pdev, nvec, nvec, PCI_IRQ_ALL_TYPES);
if (ret < 0)
        goto out_err;
```

上述请求类型的最臭名昭著的例子是为设备启用单个 MSI 模式。这可以通过将两个 1 作为 “min_vecs” 和 “max_vecs” 传递来完成：

```c
ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_ALL_TYPES);
if (ret < 0)
        goto out_err;
```

有些设备可能不支持使用传统线路中断，在这种情况下，驱动程序可以指定只有 MSI 或 MSI-X 是可接受的：

```c
nvec = pci_alloc_irq_vectors(pdev, 1, nvec, PCI_IRQ_MSI | PCI_IRQ_MSIX);
if (nvec < 0)
        goto out_err;
```

4.4.3. 旧式 API

以下用于启用和禁用 MSI 或 MSI-X 中断的旧 API 不应在新代码中使用：

```
pci_enable_msi()              /* 已废弃 */
pci_disable_msi()             /* 已废弃 */
pci_enable_msix_range()       /* 已废弃 */
pci_enable_msix_exact()       /* 已废弃 */
pci_disable_msix()            /* 已废弃 */
```

此外，还有一些 API 可以提供支持的 MSI 或 MSI-X 向量的数量：[`pci_msi_vec_count()`](https://docs.kernel.org/driver-api/pci/pci.html#c.pci_msi_vec_count) 和 [`pci_msix_vec_count()`](https://docs.kernel.org/PCI/msi-howto.html#c.pci_msix_vec_count) 。一般来说，应该避免这些情况，这有利于让 [`pci_alloc_irq_vectors()`](https://docs.kernel.org/PCI/msi-howto.html#c.pci_alloc_irq_vectors) 限制向量的数量。如果您对向量计数有特殊且合理的使用情况，我们可能必须重新审视该决定，并添加一个 pci_nr_irq_vectors() 辅助函数来透明地处理 MSI 和 MSI-X 。

### 4.4.4. 使用 MSI 时的考虑事项

#### 4.4.4.1. 自旋锁

在大多数设备驱动程序中，每个设备都有一个自旋锁，这个自旋锁是在中断处理程序中获取的。对于基于引脚的中断或单个 MSI，不需要禁用中断（Linux 保证相同的中断不会重入）。如果设备使用多个中断，驱动程序必须在持有锁时禁用中断。如果设备发送不同的中断，驱动程序将在尝试递归获取自旋锁时死锁。通过使用 spin_lock_irqsave() 或 spin_lock_irq() 可以避免这样的死锁，这两个函数禁用本地中断并获取锁（见[不可靠的锁指南](https://docs.kernel.org/kernel-hacking/locking.html)）。

#### 4.4.5. 如何判断设备上是否启用了 MSI/MSI-X

使用 “lspci -v”（作为 root）可能会显示一些带有 "MSI"、"消息信号中断"或 "MSI-X" 功能的设备。这些功能中的每一个都有一个 “Enable” 标志，其后是 "+"（已启用）或 "-"（已禁用）。

## 4.5. MSI 异常情况

已知有几个 PCI 芯片组或设备不支持 MSI。PCI 栈提供了三种禁用 MSI 的方法：

1. 全局
2. 在特定桥后面的所有设备上
3. 在单个设备上

### 4.5.1. 全局禁用 MSI

有些主机芯片组根本无法正确支持 MSI 。如果幸运的话，制造商知道这一点，并已在 ACPI FADT 表中注明。在这种情况下，Linux 会自动禁用 MSI。有些主板没有在表中包含这些信息，所以我们必须自己检测它们。它们的完整列表可以在 drivers/pci/quicks.c 中的 quick_disable_all_msi() 函数附近找到。

如果您有一块存在 MSI 问题的主板，您可以在内核命令行中添加 pci=nomsi 来禁用所有设备上的 MSI 。为了您的最佳利益，请将问题报告给 linux-pci@vger.kernel.org，包括完整的 “lspci -v”，以便我们可以将异常情况添加到内核中。

### 4.5.2 在桥后面禁用 MSI

某些 PCI 桥接器无法正确地在总线之间路由 MSI 。在这种情况下，必须在桥接器后面的所有设备上禁用 MSI 。

一些桥接器允许您通过更改它们的 PCI 配置空间中的一些位来启用 MSI（特别是 Hypertransport 芯片组，如 Nvidia nForce 和 Serverworks HT2000）。与主机芯片组一样，Linux 大多数情况下都知道它们，并且如果可以的话会自动启用 MSI 。如果您有一个 Linux 未知的桥接器，则可以使用您知道有效的任何方法在配置空间中启用 MSI，然后通过执行以下操作在该桥接器上启用 MSI：

```shell
echo 1 > /sys/bus/pci/devices/$bridge/msi_bus
```

其中 $bridge 是您启用的桥接器的 PCI 地址（例如 0000:00:0e.0）。

要禁用 MSI，请使用 1 而不是 0 。更改此值应谨慎进行，因为它可能会破坏此桥接器下所有设备的中断处理。

再次，请通知 linux-pci@vger.kernel.org 任何需要特殊处理的桥接器。

### 4.5.3. 在单个设备上禁用 MSI

已知某些设备具有错误的 MSI 实现。通常这在单独的设备驱动程序中处理，但偶尔需要使用异常情况处理。某些驱动程序可以选择禁用MSI。虽然这对驱动程序作者来说是一个方便的解决方法，但这不是一个好的做法，不应该被模仿。

### 4.5.4. 查找设备上禁用 MSI 的原因

从以上三个部分可以看出，给定设备可能无法启用 MSI 的原因有很多。您的第一步应该是仔细检查 dmesg，以确定机器是否已启用 MSI 。您还应该检查您的 .config 以确保已启用 CONFIG_PCI_MSI 。

然后，“lspci -t” 给出了一个设备上面的桥接器列表。读取 */sys/bus/pci/devices/\*/msi_bus* 将告诉您 MSI 是否已启用（1）或已禁用（0）。如果在 PCI 根和设备之间的任何桥接器的 msi_bus 文件中发现 0，则 MSI 已禁用。

还值得检查设备驱动程序，看看它是否支持 MSI 。例如，它可能包含对 [`pci_alloc_irq_vectors()`](https://docs.kernel.org/PCI/msi-howto.html#c.pci_alloc_irq_vectors) 的调用，其中包含 PCI_IRQ_MSI 或 PCI_IRQ_MSIX 标志。

## 4.6. 设备驱动程序 MSI(-X) API 列表

PCI/MSI 子系统有一个专门的 C 文件用于其导出的设备驱动程序 API —— *drivers/pci/msi/api.c* 。以下函数被导出：

> 译者注：此处不翻译 API 内容，有关 API 参数、描述、注意、返回值等信息可以查看[原文](https://docs.kernel.org/PCI/msi-howto.html#list-of-device-drivers-msi-x-apis)。另外，较新的内核版本（比如 5.4）已经不同，文件已修改为 *drivers/pci/msi.c* ，且部分函数已弃用。

| 函数                                                         | 功能                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| int **pci_enable_msi**(struct pci_dev *dev)                  | 在设备上启用 MSI 中断模式                                    |
| void **pci_disable_msi**(struct pci_dev *dev)                | 在设备上禁用 MSI 中断模式                                    |
| int **pci_msix_vec_count**(struct pci_dev *dev)              | 获取设备上 MSI-X 中断向量的数量                              |
| int **pci_enable_msix_range**(struct pci_dev *dev, struct msix_entry *entries, int minvec, int maxvec) | 在设备上启用 MSI-X 中断模式                                  |
| bool **pci_msix_can_alloc_dyn**(struct pci_dev *dev)         | 查询是否支持启用 MSI-X 后的动态分配                          |
| struct msi_map **pci_msix_alloc_irq_at**(struct pci_dev *dev, unsigned int index, const struct [irq_affinity_desc](https://docs.kernel.org/core-api/genericirq.html#c.irq_affinity_desc) *affdesc) | 在启用给定的 MSI-X 向量索引或任何空闲向量索引后分配一个 MSI-X 中断 |
| void **pci_msix_free_irq**(struct pci_dev *dev, struct msi_map map) | 释放 PCI/MSI-X 中断域上的中断                                |
| void **pci_disable_msix**(struct pci_dev *dev)               | 在设备上禁用 MSI-X 中断模式                                  |
| int **pci_alloc_irq_vectors**(struct pci_dev *dev, unsigned int min_vecs, unsigned int max_vecs, unsigned int flags) | 分配多个设备中断向量                                         |
| int **pci_alloc_irq_vectors_affinity**(struct pci_dev *dev, unsigned int min_vecs, unsigned int max_vecs, unsigned int flags, struct [irq_affinity](https://docs.kernel.org/core-api/genericirq.html#c.irq_affinity) *affd) | 根据亲和性要求分配多个设备中断向量                           |
| int **pci_irq_vector**(struct pci_dev *dev, unsigned int nr) | 获取设备中断向量的Linux IRQ编号                              |
| const struct cpumask ***pci_irq_get_affinity**(struct pci_dev *dev, int nr) | 获取设备中断向量亲和性                                       |
| struct msi_map **pci_ims_alloc_irq**(struct pci_dev *dev, union msi_instance_cookie *icookie, const struct [irq_affinity_desc](https://docs.kernel.org/core-api/genericirq.html#c.irq_affinity_desc) *affdesc) | 在 PCI/IMS 中断域上分配中断                                  |
| void **pci_ims_free_irq**(struct pci_dev *dev, struct msi_map map) | 在通过 pci_ims_alloc_irq() 分配的 PCI/IMS 中断域上分配一个中断 |
| void **pci_free_irq_vectors**(struct pci_dev *dev)           | 为设备释放之前分配的 IRQ                                     |
| void **pci_restore_msi_state**(struct pci_dev *dev)          | 在设备上恢复缓存的 MSI(-X) 状态                              |
| int **pci_msi_enabled**(void)                                | 查询 MSI(-X) 中断是否在系统范围内启用                        |
