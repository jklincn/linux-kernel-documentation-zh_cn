> 原文：[Dynamic DMA mapping Guide — The Linux Kernel documentation](https://docs.kernel.org/core-api/dma-api-howto.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.07.27
>
> 修订：

# 动态 DMA 映射指南

这是一份给设备驱动程序编写者的指南，通过示例伪代码说明了如何使用 DMA API。关于 API 的简洁描述，请参见[使用通用设备的动态 DMA 映射](dma-api.md)。

## CPU 和 DMA 地址

在 DMA API 中涉及几种类型的地址，理解它们之间的区别很重要。

内核通常使用虚拟地址。通过 [`kmalloc()`](https://docs.kernel.org/core-api/mm-api.html#c.kmalloc)、[`vmalloc()`](https://docs.kernel.org/core-api/mm-api.html#c.vmalloc) 以及类似接口返回的任何地址都是虚拟地址，可以存储在 `void*` 中。

虚拟内存系统（TLB、页表等）将虚拟地址转换为 CPU 物理地址，这些地址存储为 "phys_addr_t" 或 "resource_size_t" 。内核将寄存器等设备资源作为物理地址进行管理。这些地址在 /proc/iomem 中。物理地址对驱动程序没有直接的用处；它必须使用 [`ioremap()`](https://docs.kernel.org/driver-api/device-io.html#c.ioremap) 来映射空间并生成一个虚拟地址。

I/O 设备使用第三种地址：“总线地址”。如果设备在 MMIO 地址上有寄存器，或者它执行 DMA 来读取或写入系统内存，设备使用的地址就是总线地址。在某些系统中，总线地址与 CPU 物理地址相同，但通常它们不相同。IOMMU 和主桥可以在物理地址和总线地址之间产生任意映射。

从设备的角度看，DMA 使用的是总线地址空间，但可能仅限于该空间的一个子集。例如，即使一个系统支持 64 位的内存和 PCI BAR 地址，它也可能使用 IOMMU，这样设备只需要使用 32 位 DMA 地址。

下面是一幅图和一些示例：

```
             CPU                  CPU                  Bus
           Virtual              Physical             Address
           Address              Address               Space
            Space                Space

          +-------+             +------+             +------+
          |       |             |MMIO  |   Offset    |      |
          |       |  Virtual    |Space |   applied   |      |
        C +-------+ --------> B +------+ ----------> +------+ A
          |       |  mapping    |      |   by host   |      |
+-----+   |       |             |      |   bridge    |      |   +--------+
|     |   |       |             +------+             |      |   |        |
| CPU |   |       |             | RAM  |             |      |   | Device |
|     |   |       |             |      |             |      |   |        |
+-----+   +-------+             +------+             +------+   +--------+
          |       |  Virtual    |Buffer|   Mapping   |      |
        X +-------+ --------> Y +------+ <---------- +------+ Z
          |       |  mapping    | RAM  |   by IOMMU
          |       |             |      |
          |       |             |      |
          +-------+             +------+
```

在枚举过程中，内核发现 I/O 设备及其 MMIO 空间以及将它们连接到系统的主桥。例如，如果 PCI 设备有一个 BAR，内核会从 BAR 读取总线地址（A）并将其转换为 CPU 物理地址（B）。地址 B 存储在资源结构体（struct resource）中，通常通过 /proc/iomem 公开。当驱动程序声明一个设备时，它通常使用 [ioremap()](https://docs.kernel.org/driver-api/device-io.html#c.ioremap) 将物理地址 B 映射到虚拟地址（C）。然后，它可以使用例如 ioread32（C）来访问总线地址 A 处的设备寄存器。

如果设备支持 DMA，驱动程序将使用 [kmalloc()](https://docs.kernel.org/core-api/mm-api.html#c.kmalloc) 或类似接口设置缓冲区，返回虚拟地址（X）。虚拟内存系统将 X 映射到系统 RAM 中的物理地址（Y）。驱动程序可以使用虚拟地址 X 访问缓冲区，但设备本身不能，因为 DMA 不通过 CPU 虚拟内存系统。

在一些简单的系统中，设备可以直接对物理地址 Y 进行 DMA。但在许多其他系统中，会有 IOMMU 硬件来将 DMA 地址转换为物理地址，例如，将 Z 转换为 Y。这是 DMA API 的部分原因：驱动程序可以向 dma_map_single() 等接口提供虚拟地址 X，该接口设置任何所需的 IOMMU 映射并返回 DMA 地址 Z。然后，驱动程序告诉设备对 Z 进行 DMA，IOMMU 将其映射到系统 RAM 中地址 Y 的缓冲区。

为了使 Linux 能够使用动态 DMA 映射，它需要驱动程序的一些帮助，即必须考虑到 DMA 地址只应在实际使用时进行映射，并在 DMA 传输后取消映射。

以下 API 可以在不存在此类硬件的平台上工作。

注意，DMA API 与任何独立于底层微处理器体系结构的总线一起工作。您应该使用 DMA API，而不是特定于总线的 DMA API，即使用 DMA_map\_\*() 接口，而不是 pci_map\_\*() 接口。

首先，你应该确保在您的驱动程序中：

```c
#include <linux/dma-mapping.h>
```

它提供了 dma_addr_t 的定义。此类型可以保存平台的任何有效 DMA 地址，并且应该在您保存 DMA 映射函数返回的 DMA 地址的任何地方使用。

## 什么内存可以进行DMA操作？

首先，你必须了解哪些内核内存可以用于 DMA 映射工具。关于这一点，曾经有一套不成文的规则，而本文尝试最终将其记录下来。

如果你通过页面分配器获取内存（例如，__get_free_page*()）或者通用内存分配器（例如，[kmalloc()](https://docs.kernel.org/core-api/mm-api.html#c.kmalloc) 或 [kmem_cache_alloc()](https://docs.kernel.org/core-api/mm-api.html#c.kmem_cache_alloc)），那么你可以使用这些函数返回的地址进行 DMA 操作。

具体来说，这意味着你不能使用 vmalloc() 返回的内存/地址进行 DMA。可以对映射到 vmalloc() 区域的底层内存进行 DMA，但这需要遍历页表以获取物理地址，然后使用类似 __va() 的东西将这些页翻译回内核地址。[注：当我们整合 Gerd Knorr 的通用代码来完成这个操作时，会更新此部分内容。]

这条规则还意味着你不能使用内核映像地址（数据段/文本段/BSS段中的条目），也不能使用模块映像地址，或堆栈地址进行 DMA。这些地址可能与物理内存的其他部分完全不同。即使这些类型的内存可以物理上支持 DMA，你也需要确保 I/O 缓冲区是缓存行对齐的。否则，在具有 DMA 不一致缓存的 CPU 上，你会遇到缓存行共享问题（数据损坏）。例如，CPU 可能会写入一个字，而 DMA 会写入同一缓存行中的另一个字，这可能导致其中一个被覆盖。

此外，这意味着你不能使用 kmap() 调用返回的地址进行DMA操作。这与 vmalloc() 类似。

那么块 I/O 和网络缓冲区呢？块 I/O 和网络子系统确保它们使用的缓冲区是有效的，可以进行 DMA 操作。

## DMA 寻址能力

默认情况下，内核假定你的设备可以进行 32 位的 DMA 地址访问。对于支持 64 位的设备，这需要增加，对于有局限性的设备，这需要减少。

关于 PCI 的特别注意事项：PCI-X 规范要求 PCI-X 设备支持所有事务的 64 位地址（DAC）。并且至少有一个平台（SGI SN2）要求在 IO 总线处于 PCI-X 模式时，64 位一致性分配才能正确运行。

为了正确操作，必须设置 DMA 掩码以通知内核你的设备的 DMA 寻址能力。

这可以通过调用 dma_set_mask_and_coherent() 来完成：

```c
int dma_set_mask_and_coherent(struct device *dev, u64 mask);
```

这将同时为流映射和一致性 API 设置掩码。如果你有一些特殊要求，可以分别使用以下两个调用：

- 流映射的设置通过调用 dma_set_mask() 完成：

  ```c
  int dma_set_mask(struct device *dev, u64 mask);
  ```

- 一致性分配的设置通过调用 dma_set_coherent_mask() 完成：

  ```c
  int dma_set_coherent_mask(struct device *dev, u64 mask);
  ```

这里，dev 是指向你的设备的设备结构体的指针，mask 是一个位掩码，描述你的设备支持的地址位数。通常，你的设备的设备结构体嵌入在设备的总线特定的设备结构体中。例如，&pdev->dev 是指向 PCI 设备的设备结构体的指针（pdev 是指向你的设备的 PCI 设备结构体的指针）。

这些调用通常返回零，表示你的设备可以在给定地址掩码下正确进行 DMA 操作，但如果掩码太小以至于在给定系统上不支持，它们可能会返回错误。如果返回非零值，表示你的设备无法在该平台上正确进行 DMA 操作，尝试这样做将导致未定义的行为。除非 dma_set_mask 系列函数返回成功，否则不得在该设备上使用 DMA。

这意味着在失败的情况下，你有两个选择：

1. 使用一些非 DMA 模式进行数据传输（如果可能）。
2. 忽略此设备并不初始化它。

建议你的驱动程序在设置 DMA 掩码失败时打印一个内核 KERN_WARNING 消息。这样，如果驱动程序的用户反馈性能不佳或设备未被检测到，你可以让他们提供内核消息以确切了解原因。

具备 24 位寻址能力的设备应如下操作：

```c
if (dma_set_mask_and_coherent(dev, DMA_BIT_MASK(24))) {
        dev_warn(dev, "mydev: No suitable DMA available\n");
        goto ignore_this_device;
}
```

标准的 64 位寻址设备应如下操作：

```c
dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64))
```

dma_set_mask_and_coherent() 在使用 DMA_BIT_MASK(64) 时从不返回失败。典型的错误代码如下：

```c
/* Wrong code */
if (dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64)))
        dma_set_mask_and_coherent(dev, DMA_BIT_MASK(32))
```

dma_set_mask_and_coherent() 在使用大于 32 的值时从不会返回失败。因此，典型的代码应如下所示：

```c
/* Recommended code */
if (support_64bit)
        dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
else
        dma_set_mask_and_coherent(dev, DMA_BIT_MASK(32));
```

如果设备仅支持一致性分配中的 32 位地址，但支持流映射的完整 64 位地址，则应如下操作：

```c
if (dma_set_mask(dev, DMA_BIT_MASK(64))) {
        dev_warn(dev, "mydev: No suitable DMA available\n");
        goto ignore_this_device;
}
```

一致性掩码总是可以设置为与流掩码相同或更小的掩码。然而，对于仅使用一致性分配的设备驱动程序的罕见情况，必须检查 dma_set_coherent_mask() 的返回值。

最后，如果你的设备只能驱动低 24 位地址，你可以这样做：

```c
if (dma_set_mask(dev, DMA_BIT_MASK(24))) {
        dev_warn(dev, "mydev: 24-bit DMA addressing not available\n");
        goto ignore_this_device;
}
```

当 dma_set_mask() 或 dma_set_mask_and_coherent() 成功并返回零时，内核会保存你提供的这个掩码。内核将在你进行 DMA 映射时使用这些信息。

我们目前知道一种情况，值得在此文档中提及。如果你的设备支持多个功能（例如，声卡提供播放和录音功能），并且各种不同功能有不同的 DMA 地址限制，你可能希望探测每个掩码并仅提供机器可以处理的功能。最后一次调用 dma_set_mask() 时应使用最具体的掩码。

以下是展示如何完成此操作的伪代码：

```c
#define PLAYBACK_ADDRESS_BITS   DMA_BIT_MASK(32)
#define RECORD_ADDRESS_BITS     DMA_BIT_MASK(24)

struct my_sound_card *card;
struct device *dev;

...
if (!dma_set_mask(dev, PLAYBACK_ADDRESS_BITS)) {
        card->playback_enabled = 1;
} else {
        card->playback_enabled = 0;
        dev_warn(dev, "%s: Playback disabled due to DMA limitations\n",
               card->name);
}
if (!dma_set_mask(dev, RECORD_ADDRESS_BITS)) {
        card->record_enabled = 1;
} else {
        card->record_enabled = 0;
        dev_warn(dev, "%s: Record disabled due to DMA limitations\n",
               card->name);
}

```

这里以声卡为例，因为这类 PCI 设备似乎充斥着给 PCI 前端的 ISA 芯片，因此保留了 ISA 的 16MB DMA 地址限制。

## DMA 映射类型

有两种类型的 DMA 映射：

- 一致性 DMA 映射，通常在驱动程序初始化时映射，在结束时取消映射，硬件应保证设备和 CPU 可以并行访问数据，并且能够看到彼此的更新而无需任何显式的软件刷新。

  可以将“一致性”视为“同步”或“一致”。

  > 译者注：原文为 Think of “consistent” as “synchronous” or “coherent”.

  当前默认是返回低 32 位 DMA 空间中的一致性内存。但是，为了未来的兼容性，即使这个默认值对你的驱动程序是合适的，你也应该设置一致性掩码。

  使用一致性映射的好例子包括：

  - 网络卡的 DMA 环描述符。
  - SCSI 适配器邮箱命令数据结构。
  - 从主内存中执行的设备固件微代码。

  这些例子所需的不变条件是，任何CPU对内存的存储操作对设备立即可见，反之亦然。一致性映射保证了这一点。

  > [!IMPORTANT]
  >
  > 一致性 DMA 内存并不排除使用适当的内存屏障。CPU 可能会像对待普通内存一样重新排序对一致性内存的存储操作。例如，如果设备需要首先看到描述符的第一个字更新，然后是第二个字，你必须执行如下操作：
  >
  > ```c
  > desc->word0 = address;
  > wmb();
  > desc->word1 = DESC_VALID;
  > ```
  >
  > 这样才能在所有平台上获得正确的行为。
  >
  > 此外，在某些平台上，驱动程序可能需要刷新 CPU 写缓冲区，就像需要刷新 PCI 桥中的写缓冲区一样（例如，在写入寄存器值后读取该寄存器的值）。

- 流式DMA映射，通常为一次 DMA 传输映射，之后立即取消映射（除非你使用下面的 dma_sync_* ），硬件可以针对顺序访问进行优化。

  可以将“流式”视为“异步”或“在一致性域之外”。

  使用流式映射的好例子包括：

  - 设备传输/接收的网络缓冲区。
  - SCSI 设备写入/读取的文件系统缓冲区。

  使用这种类型映射的接口设计使实现可以进行硬件允许的任何性能优化。为此，当使用这种映射时，你必须明确你想要发生什么。

这两种 DMA 映射类型都没有来自底层总线的对齐限制，尽管某些设备可能有这种限制。此外，对于非 DMA 一致性的缓存系统，当底层缓冲区不与其他数据共享缓存行时，工作效果会更好。

## 使用一致性DMA映射

要分配和映射大的（PAGE_SIZE 左右）一致性 DMA 区域，你应该这样做：

```c
dma_addr_t dma_handle;

cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, gfp);
```

其中，dev 是一个`struct device *`。这可以在中断上下文中使用 GFP_ATOMIC 标志调用。

size 是你要分配的区域的长度，以字节为单位。

这个函数将为该区域分配 RAM，因此它的行为类似于 __get_free_pages() （但使用大小而不是页数）。如果你的驱动程序需要小于一页的区域，你可能更喜欢使用下面描述的 dma_pool 接口。

一致性 DMA 映射接口默认返回一个 32 位可寻址的 DMA 地址。即使设备通过 DMA 掩码表明它可以访问大于 32 位的地址，一致性分配只有在通过 dma_set_coherent_mask() 显式更改一致性 DMA 掩码的情况下，才会返回大于 32 位的 DMA 地址。dma_pool 接口也是如此。

dma_alloc_coherent() 返回两个值：你可以从 CPU 访问它的虚拟地址和你传递给卡的 dma_handle。

CPU 虚拟地址和 DMA 地址都保证对齐到大于或等于请求大小的最小 PAGE_SIZE 顺序。这一不变性存在（例如）以保证如果你分配的块小于或等于 64 千字节，你收到的缓冲区范围不会跨越 64K 边界。

要取消映射并释放这样的 DMA 区域，你调用：

```c
dma_free_coherent(dev, size, cpu_addr, dma_handle);
```

其中 dev ，size 与上述调用中的相同，cpu_addr 和 dma_handle 是 dma_alloc_coherent() 返回给你的值。这个函数不能在中断上下文中调用。

如果你的驱动程序需要许多较小的内存区域，你可以编写自定义代码来细分 dma_alloc_coherent() 返回的页，或者你可以使用 dma_pool API 来完成这一点。dma_pool 类似于 kmem_cache，但它使用dma_alloc_coherent()，而不是 __get_free_pages()。此外，它理解常见的硬件对齐约束，例如队列头需要在 N 字节边界上对齐。

这样创建一个 dma_pool：

```c
struct dma_pool *pool;

pool = dma_pool_create(name, dev, size, align, boundary);
```

name 用于诊断（类似 kmem_cache name）； dev 和 size 如上所述。设备对此类数据的硬件对齐要求是 align（以字节为单位，必须是 2 的幂）。如果你的设备没有边界跨越限制，传递 0 作为边界；传递 4096 表示从此池中分配的内存不得跨越 4K 字节边界（但那时可能最好直接使用 dma_alloc_coherent()）。

从 DMA 池中分配内存如下：

```c
cpu_addr = dma_pool_alloc(pool, flags, &dma_handle);
```

如果允许阻塞（不在中断上下文中或不持有 SMP 锁），flags 为 GFP_KERNEL，否则为 GFP_ATOMIC。像 dma_alloc_coherent() 一样，这返回两个值，cpu_addr 和 dma_handle。

释放从 dma_pool 分配的内存如下：

```c
dma_pool_free(pool, cpu_addr, dma_handle);
```

其中 pool 是你传递给 [dma_pool_alloc()](https://docs.kernel.org/core-api/mm-api.html#c.dma_pool_alloc) 的，cpu_addr 和 dma_handle 是 [dma_pool_alloc()](https://docs.kernel.org/core-api/mm-api.html#c.dma_pool_alloc) 返回的值。这个函数可以在中断上下文中调用。

通过调用销毁 dma_pool：

```c
 dma_pool_destroy(pool) 
```

在销毁池之前，确保你已经为池中分配的所有内存调用了 dma_pool_free()。这个函数不能在中断上下文中调用。

## DMA方向

本文档后续部分描述的接口需要一个 DMA 方向参数，这是一个整数，取以下值之一：

```
DMA_BIDIRECTIONAL
DMA_TO_DEVICE
DMA_FROM_DEVICE
DMA_NONE
```

如果你知道确切的 DMA 方向，你应该提供这个方向。

DMA_TO_DEVICE 表示“从主存到设备”；DMA_FROM_DEVICE 表示“从设备到主存” 。这表示在 DMA 传输期间数据移动的方向。

强烈建议尽可能精确地指定这个方向。

如果你完全无法确定 DMA 传输的方向，请指定 DMA_BIDIRECTIONAL 。这意味着 DMA 可以双向进行。平台保证你可以合法地指定这一点，并且它会工作，但这可能会以性能为代价。

值 DMA_NONE 用于调试。可以在你确定确切方向之前将其保存在数据结构中，这有助于捕捉方向跟踪逻辑未能正确设置的情况。

精确指定此值的另一个优点（除了可能的特定平台优化外）是用于调试。一些平台实际上有一个写入权限布尔值，可以用来标记 DMA 映射，就像用户程序地址空间中的页保护一样。当 DMA 控制器硬件检测到权限设置违规时，这些平台可以并且确实会在内核日志中报告错误。

只有流式映射指定方向，一致性映射隐含具有 DMA_BIDIRECTIONAL 的方向属性设置。

SCSI 子系统在你的驱动程序正在处理的 SCSI 命令的 sc_data_direction 成员中告诉你要使用的方向。

对于网络驱动程序，这相当简单。对于发送数据包，使用 DMA_TO_DEVICE 方向说明符进行映射/取消映射。对于接收数据包，恰恰相反，使用 DMA_FROM_DEVICE 方向说明符进行映射/取消映射。

## 使用流式 DMA 映射

流式 DMA 映射函数可以在中断上下文中调用。每个映射/取消映射操作都有两个版本，一个用于映射/取消映射单个内存区域，一个用于映射/取消映射散列表。

要映射单个区域，你可以这样做：

```c
struct device *dev = &my_dev->dev;
dma_addr_t dma_handle;
void *addr = buffer->ptr;
size_t size = buffer->len;

dma_handle = dma_map_single(dev, addr, size, direction);
if (dma_mapping_error(dev, dma_handle)) {
    /*
     * 减少当前 DMA 映射使用，
     * 延迟并稍后重试或
     * 重置驱动程序。
     */
    goto map_error_handling;
}
```

要取消映射它：

```c
dma_unmap_single(dev, dma_handle, size, direction);
```

你应该调用 dma_mapping_error()，因为 dma_map_single() 可能会失败并返回错误。这样做可以确保映射代码在所有 DMA 实现中都能正常工作，而不依赖于底层实现的具体细节。使用返回的地址而不检查错误可能导致从崩溃到数据静默损坏的各种故障。这同样适用于 dma_map_page()。

当 DMA 活动结束时，例如在中断告诉你 DMA 传输完成时，你应该调用 dma_unmap_single()。

像这样使用 CPU 指针进行单次映射有一个缺点：你不能以这种方式引用高内存（HIGHMEM）。因此，有一对类似于 dma\_{map,unmap}\_single() 的映射/取消映射接口。这些接口处理页/偏移对，而不是 CPU 指针。具体如下：

```c
struct device *dev = &my_dev->dev;
dma_addr_t dma_handle;
struct page *page = buffer->page;
unsigned long offset = buffer->offset;
size_t size = buffer->len;

dma_handle = dma_map_page(dev, page, offset, size, direction);
if (dma_mapping_error(dev, dma_handle)) {
    /*
     * 减少当前 DMA 映射使用，
     * 延迟并稍后重试或
     * 重置驱动程序。
     */
    goto map_error_handling;
}

...

dma_unmap_page(dev, dma_handle, size, direction);
```

这里，offset 是指给定页内的字节偏移量。

你应该调用 dma_mapping_error()，因为 dma_map_page()可能会失败并返回错误，正如在 dma_map_single() 讨论中所述。

当 DMA 活动结束时，例如在中断告诉你 DMA 传输完成时，你应该调用 dma_unmap_page()。

对于散列表，你可以通过以下方式映射从多个区域收集的区域：

```c
int i, count = dma_map_sg(dev, sglist, nents, direction);
struct scatterlist *sg;

for_each_sg(sglist, sg, count, i) {
    hw_address[i] = sg_dma_address(sg);
    hw_len[i] = sg_dma_len(sg);
}
```

其中，nents 是 sglist 中的条目数量。

实现可以将几个连续的 sglist 条目合并为一个（例如，如果 DMA 映射以 PAGE_SIZE 粒度进行，只要第一个条目结束和第二个条目在页面边界开始，任何连续的 sglist 条目都可以合并为一个——事实上，这对于不能进行散列收集或具有非常有限的散列收集条目的卡来说是一个巨大的优势），并返回它们映射到的实际 sg 条目数。失败时返回 0。

然后你应该循环 count 次（注意：这可能少于 nents 次）并使用 sg_dma_address() 和 sg_dma_len() 宏，如上所示，而不是之前访问的 sg->address 和 sg->length。

要取消映射散列表，只需调用：

```c
dma_unmap_sg(dev, sglist, nents, direction);
```

再次确保 DMA 活动已经结束。

> [!NOTE]
>
> 传递给 dma_unmap_sg 调用的 nents 参数必须与你传递给 dma_map_sg 调用的参数相同，它不应是 dma_map_sg 调用返回的 count 值。

每个 dma\_map\_{single,sg}() 调用都应有其对应的 dma\_unmap\_{single,sg}() 调用，因为 DMA 地址空间是一个共享资源，你可能会通过消耗所有 DMA 地址使机器不可用。

如果你需要多次使用相同的流式 DMA 区域并在 DMA 传输之间处理数据，则需要正确同步缓冲区，以便 CPU 和设备看到最新和正确的 DMA 缓冲区副本。

所以，首先，只需使用 dma\_map\_{single,sg}() 进行映射，并在每次 DMA 传输后调用：

```c
dma_sync_single_for_cpu(dev, dma_handle, size, direction);
```

或：

```c
dma_sync_sg_for_cpu(dev, sglist, nents, direction);
```

然后，如果你希望设备再次访问 DMA 区域，在 CPU 完成对数据的访问后，并且在实际将缓冲区交给硬件之前调用：

```c
dma_sync_single_for_device(dev, dma_handle, size, direction);
```

或：

```c
dma_sync_sg_for_device(dev, sglist, nents, direction);
```

> [!NOTE]
>
> 传递给 dma_sync_sg_for_cpu() 和 dma_sync_sg_for_device() 的 nents 参数必须是传递给 dma_map_sg() 的相同参数。它不是 dma_map_sg() 返回的 count。

在最后一次 DMA 传输后调用一个 DMA 取消映射例程 dma_unmap\_{single,sg}()。如果从第一次 dma\_map\_\*() 调用到 dma_unmap_\*() 调用之间不处理数据，则不需要调用 dma_sync\_\*() 函数。

以下是展示你需要使用 dma_sync_*() 接口的情况的伪代码：

```c
my_card_setup_receive_buffer(struct my_card *cp, char *buffer, int len)
{
        dma_addr_t mapping;

        mapping = dma_map_single(cp->dev, buffer, len, DMA_FROM_DEVICE);
        if (dma_mapping_error(cp->dev, mapping)) {
                /*
                 * 减少当前的 DMA 映射使用，
                 * 延迟并稍后重试或
                 * 重置驱动程序。
                 */
                goto map_error_handling;
        }

        cp->rx_buf = buffer;
        cp->rx_len = len;
        cp->rx_dma = mapping;

        give_rx_buf_to_card(cp);
}

...

my_card_interrupt_handler(int irq, void *devid, struct pt_regs *regs)
{
        struct my_card *cp = devid;

        ...
        if (read_card_status(cp) == RX_BUF_TRANSFERRED) {
                struct my_card_header *hp;

                /* 检查头部以确定是否
                 * 接受数据。但首先要同步
                 * DMA 传输与 CPU，以便我们看到更新的内容。
                 */
                dma_sync_single_for_cpu(&cp->dev, cp->rx_dma,
                                        cp->rx_len,
                                        DMA_FROM_DEVICE);

                /* 现在可以安全地检查缓冲区了。 */
                hp = (struct my_card_header *) cp->rx_buf;
                if (header_is_ok(hp)) {
                        dma_unmap_single(&cp->dev, cp->rx_dma, cp->rx_len,
                                         DMA_FROM_DEVICE);
                        pass_to_upper_layers(cp->rx_buf);
                        make_and_setup_new_rx_buf(cp);
                } else {
                        /* CPU 不应写入
                         * DMA_FROM_DEVICE 映射的区域，
                         * 因此这里不需要 dma_sync_single_for_device()。
                         * 如果是 DMA_BIDIRECTIONAL 映射，
                         * 并且内存被修改，则需要此操作。
                         */
                        give_rx_buf_to_card(cp);
                }
        }
}

```

## 错误处理

在某些体系结构上，DMA 地址空间是有限的，可以通过以下方式确定分配失败：

- 检查 dma_alloc_coherent() 是否返回 NULL 或 dma_map_sg 是否返回 0。

- 使用 dma_mapping_error() 检查从 dma_map_single() 和 dma_map_page() 返回的 dma_addr_t：

  ```c
  dma_addr_t dma_handle;
  
  dma_handle = dma_map_single(dev, addr, size, direction);
  if (dma_mapping_error(dev, dma_handle)) {
      /*
       * 减少当前 DMA 映射使用，
       * 延迟并稍后重试或
       * 重置驱动程序。
       */
      goto map_error_handling;
  }
  ```

- 在多页映射尝试中，如果在中间发生映射错误，取消已经映射的页。这些示例也适用于dma_map_page()。

示例 1：

```c
dma_addr_t dma_handle1;
dma_addr_t dma_handle2;

dma_handle1 = dma_map_single(dev, addr, size, direction);
if (dma_mapping_error(dev, dma_handle1)) {
        /*
         * 减少当前 DMA 映射使用，
         * 延迟并稍后重试或
         * 重置驱动程序。
         */
        goto map_error_handling1;
}
dma_handle2 = dma_map_single(dev, addr, size, direction);
if (dma_mapping_error(dev, dma_handle2)) {
        /*
         * 减少当前 DMA 映射使用，
         * 延迟并稍后重试或
         * 重置驱动程序。
         */
        goto map_error_handling2;
}

...

map_error_handling2:
        dma_unmap_single(dma_handle1);
map_error_handling1:
```

示例 2：

```c
/*
 * 如果在循环中分配缓冲区，当在中间检测到映射错误时，
 * 取消映射所有已映射的缓冲区
 */

dma_addr_t dma_addr;
dma_addr_t array[DMA_BUFFERS];
int save_index = 0;

for (i = 0; i < DMA_BUFFERS; i++) {

        ...

        dma_addr = dma_map_single(dev, addr, size, direction);
        if (dma_mapping_error(dev, dma_addr)) {
                /*
                 * 减少当前 DMA 映射使用，
                 * 延迟并稍后重试或
                 * 重置驱动程序。
                 */
                goto map_error_handling;
        }
        array[i].dma_addr = dma_addr;
        save_index++;
}

...

map_error_handling:

for (i = 0; i < save_index; i++) {

        ...

        dma_unmap_single(array[i].dma_addr);
}
```

网络驱动程序必须调用 dev_kfree_skb() 来释放套接字缓冲区，并在传输挂钩（ndo_start_xmit）中 DMA 映射失败时返回 NETDEV_TX_OK 。这意味着在失败的情况下，套接字缓冲区会被丢弃。

SCSI 驱动程序必须在队列命令挂钩中 DMA 映射失败时返回 SCSI_MLQUEUE_HOST_BUSY。这意味着 SCSI 子系统稍后会再次将命令传递给驱动程序。

## 优化取消映射状态空间消耗

在许多平台上，dma_unmap_{single,page}() 只是一个空操作。因此，跟踪映射地址和长度是浪费空间的。为了避免在驱动程序中充满条件编译和类似的“变通方法”（这会破坏可移植 API 的整个目的），提供了以下方法。

实际上，与其一个个描述宏，不如通过转换一些示例代码来说明。

在状态保存结构中使用 DEFINE_DMA_UNMAP_{ADDR,LEN}。例如，修改前：

```c
struct ring_state {
        struct sk_buff *skb;
        dma_addr_t mapping;
        __u32 len;
};
```

修改后：

```c
struct ring_state {
        struct sk_buff *skb;
        DEFINE_DMA_UNMAP_ADDR(mapping);
        DEFINE_DMA_UNMAP_LEN(len);
};
```

使用 dma_unmap\_{addr,len}\_set() 设置这些值。例如，修改前：

```c
ringp->mapping = FOO;
ringp->len = BAR;
```

修改后：

```c
dma_unmap_addr_set(ringp, mapping, FOO);
dma_unmap_len_set(ringp, len, BAR);
```

使用 dma_unmap_{addr,len}() 访问这些值。例如，修改前：

```c
dma_unmap_single(dev, ringp->mapping, ringp->len, DMA_FROM_DEVICE);
```

修改后：

```c
dma_unmap_single(dev,
                 dma_unmap_addr(ringp, mapping),
                 dma_unmap_len(ringp, len),
                 DMA_FROM_DEVICE);
```

这些内容应该是自解释的。我们将 ADDR 和 LEN 分开处理，因为实现可能只需要地址来执行取消映射操作。

## 平台问题

如果你只为 Linux 编写驱动程序，并且不维护内核的架构端口，可以安全地跳到“结尾”部分。

1. 散列表结构体要求

   如果架构支持 IOMMUs（包括软件 IOMMU），你需要启用 CONFIG_NEED_SG_DMA_LENGTH。

2. ARCH_DMA_MINALIGN

   架构必须确保 kmalloc 的缓冲区是 DMA 安全的。驱动程序和子系统依赖于此。如果架构不是完全 DMA 一致的（即硬件不能确保 CPU 缓存中的数据与主存中的数据相同），必须设置 ARCH_DMA_MINALIGN，以确保内存分配器保证 kmalloc 的缓冲区不与其他缓存行共享。参见 arch/arm/include/asm/cache.h 作为示例。

   注意，ARCH_DMA_MINALIGN 是关于 DMA 内存对齐约束的。你不需要担心架构的数据对齐约束（例如，关于 64 位对象的对齐约束）。

## 结尾

本文档及其 API 本身，如果没有众多个人的反馈和建议，不会形成目前的形式。我们特别提到以下人员，排名不分先后：

```
Russell King <rmk@arm.linux.org.uk>
Leo Dagum <dagum@barrel.engr.sgi.com>
Ralf Baechle <ralf@oss.sgi.com>
Grant Grundler <grundler@cup.hp.com>
Jay Estabrook <Jay.Estabrook@compaq.com>
Thomas Sailer <sailer@ife.ee.ethz.ch>
Andrea Arcangeli <andrea@suse.de>
Jens Axboe <jens.axboe@oracle.com>
David Mosberger-Tang <davidm@hpl.hp.com>
```
