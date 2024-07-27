> 原文：[Dynamic DMA mapping using the generic device — The Linux Kernel documentation](https://docs.kernel.org/core-api/dma-api.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.21
>
> 修订：

# 使用通用设备的动态 DMA 映射

本文档描述了 DMA API 。想要更轻松地了解这个 API（以及实际示例），请参见[动态 DMA 映射指南](dma-api-howto.md)。

这个 API 被分为两部分。第一部分描述了基本的 API 。第二部分描述了用于支持非一致性内存机器的扩展。除非你确信你的驱动必须支持非一致性平台（这通常只适用于遗留平台），否则你应该只使用第一部分中描述的 API 。

## 第一部分 —— dma_API

要获取 dma_API，你必须 #include<linux/dma-mapping.h> 。这将提供 dma_addr_t 和下面描述的接口。

dma_addr_t 可以持有平台上任何有效的 DMA 地址。它可以给设备用作 DMA 源或目标。CPU 不能直接引用 dma_addr_t，因为它的物理地址空间和 DMA 地址空间之间可能存在转换。

### 第一部分 a —— 使用大型 DMA 一致性缓冲区

```c
void *dma_alloc_coherent(struct device *dev, size_t size,
                   dma_addr_t *dma_handle, gfp_t flag)
```

一致性内存是指设备或处理器写入的内容可以立即被处理器或设备读取的内存，而不必担心缓存效应。（然而，你可能需要确保在告知设备读取该内存之前，先刷新处理器的写缓冲区。）

这个函数分配一段 \<size\> 字节的一致性内存。

它返回分配区域的指针（在处理器的虚拟地址空间中），如果分配失败，则返回 NULL 。

它还返回一个 \<dma_handle\> ，它可以被转换为与总线宽度相同的无符号整数，并作为该区域的 DMA 基地址提供给设备。

注意：在某些平台上，一致性内存可能会很昂贵，并且最小分配长度可能与一个页面一样大，因此你应该尽可能合并对一致性内存的请求。最简单的方法是使用 dma_pool 调用（见下文）。

flag 参数（仅限 dma_alloc_coherent()）允许调用者为分配指定 GFP_  标志（参见[`kmalloc()`](https://docs.kernel.org/core-api/mm-api.html#c.kmalloc)）（实现可能选择忽略影响返回内存位置的标志，如 GFP_DMA）。

```c
void dma_free_coherent(struct device *dev, size_t size, void *cpu_addr,
                  dma_addr_t dma_handle)
```

释放先前分配的一致性内存区域。dev、size 和 dma_handle 必须与传递给 dma_alloc_coherent() 的相同。cpu_addr 必须是 dma_alloc_coherent() 返回的虚拟地址。

请注意，与它们的同级分配调用不同，这些函数只能在启用 IRQ 的情况下调用。

### 第一部分 b —— 使用小型 DMA 一致性缓冲区

要获取 dma_API 的这一部分，你必须 #include<linux/dmapool.h> 。

许多驱动程序需要大量的小型 DMA 一致性内存区域用于 DMA 描述符或 I/O 缓冲区。与使用 dma_alloc_coherent() 以页面或更大单位分配不同，你可以使用 DMA 池。它们的工作方式很像 kmem_cache 这个结构体，只是它们使用 DMA 一致性分配器，而不是 __get_free_pages() 。另外，它们了解常见的硬件对齐限制，例如队列头需要在 N 字节边界上对齐。

```c
struct dma_pool *dma_pool_create(const char *name, struct device *dev,
                size_t size, size_t align, size_t alloc);
```

[`dma_pool_create()`](https://docs.kernel.org/core-api/mm-api.html#c.dma_pool_create) 为给定设备初始化一个 DMA 一致性缓冲区池。它必须在可以睡眠的上下文中调用。

“name” 用于诊断（类似于结构体 kmem_cache 的 name 字段）；dev 和 size 与你传递给 dma_alloc_coherent() 的相似。"align" 是这类数据的设备硬件对齐要求（以字节为单位，并且必须是 2 的幂）。如果你的设备没有越界限制，请为 alloc 传递 0；传递 4096 表示从这个池分配的内存不能跨越 4K 字节的边界。

```c
void *dma_pool_zalloc(struct dma_pool *pool, gfp_t mem_flags,
                dma_addr_t *handle)
```

封装了 [`dma_pool_alloc()`](https://docs.kernel.org/core-api/mm-api.html#c.dma_pool_alloc) ，如果分配成功，还会将返回的内存区域清零。

```c
void *dma_pool_alloc(struct dma_pool *pool, gfp_t gfp_flags,
               dma_addr_t *dma_handle);
```

这将从池中分配内存；返回的内存将满足创建时指定的大小和对齐要求。传递 GFP_ATOMIC 以防止阻塞，或者如果允许（不在中断中，不持有 SMP 锁），传递 GFP_KERNEL 以允许阻塞。与 dma_alloc_coherent() 一样，它会返回两个值：CPU 可用的地址和池的设备可用的 DMA 地址。

```c
void dma_pool_free(struct dma_pool *pool, void *vaddr,
              dma_addr_t addr);
```

这会将内存重新放入池中。pool 是之前传递给 dma_pool_alloc() 的；vaddr （CPU 地址）和 addr （DMA 地址）都是之前内存分配时返回的。

```c
void dma_pool_destroy(struct dma_pool *pool);
```

[`dma_pool_destroy()`](https://docs.kernel.org/core-api/mm-api.html#c.dma_pool_destroy) 释放池中的资源。它必须在可以睡眠的上下文中调用。在销毁池之前，请确保已将所有分配的内存释放回池中。

### 第一部分 c —— DMA 地址限制

```c
int dma_set_mask_and_coherent(struct device *dev, u64 mask)
```

检查掩码是否是可能的，并且如果可能的话，更新设备流式 DMA 和一致性 DMA 掩码参数。

> 译者注：即检查设备是否能支持这样的内存地址范围。换句话说，它是在询问是否可以将设备的 DMA 操作限制在指定的地址范围内。如果设备的硬件或驱动程序能够处理这个指定的地址掩码，则这个掩码就是“可能的”。如果设备无法支持这个掩码，则会返回一个错误，表示这个掩码不可能实现或不被支持。

返回值：如果成功则返回 0 ，如果不成功则返回表示错误的负数。

```c
int dma_set_mask(struct device *dev, u64 mask)
```

检查掩码是否可能，并且如果可能的话，更新设备参数。

返回值：如果成功则返回 0 ，如果不成功则返回表示错误的负数。

```c
int dma_set_coherent_mask(struct device *dev, u64 mask)
```

检查掩码是否可能，并且如果可能的话，更新设备参数。

返回值：如果成功则返回 0 ，如果不成功则返回表示错误的负数。

```c
u64 dma_get_required_mask(struct device *dev)
```

这个 API 返回平台为了高效操作所需的掩码。通常这意味着返回的掩码是覆盖所有内存所需的最小掩码。检查所需的掩码使描述符大小可变的驱动程序有机会在必要时使用较小的描述符。

请求所需的掩码不会改变当前掩码。如果你想利用它，你应该发出一个 dma_set_mask() 调用来将掩码设置为返回的值。

```c
size_t dma_max_mapping_size(struct device *dev)
```

返回设备的最大映射大小。映射函数的 size 参数，如 dma_map_single()、dma_map_page() 等，不应大于返回的值。

```c
size_t dma_opt_mapping_size(struct device *dev)
```

返回设备的最大最佳映射大小。

在某些情况下，映射较大的缓冲区可能需要更长的时间。此外，对于高速率的短时间的流式 DMA 映射，在映射上花费的前期时间可能占总请求寿命的相当大的一部分。因此，如果拆分较大的请求不会导致显著的性能损失，则建议设备驱动程序将流式 DMA 映射的总长度限制为该函数的返回值。

```c
bool dma_need_sync(struct device *dev, dma_addr_t dma_addr)
```

如果需要调用 dma_sync_single_for_{device, cpu} 来传输内存所有权，则返回 true 。如果可以跳过这些调用，则返回 false 。

```c
unsigned long dma_get_merge_boundary(struct device *dev);
```

返回 DMA 合并边界。如果设备不能合并任何 DMA 地址段，则函数返回 0 。

### 第一部分 d —— 流式 DMA 映射

```c
dma_addr_t dma_map_single(struct device *dev, void *cpu_addr, size_t size,
               enum dma_data_direction direction)
```

映射一块处理器虚拟内存，以便设备可以访问它，并返回内存的 DMA 地址。

这两个 API 的方向可以通过类型转换自由转换。但是，dma_API 使用一个强类型的枚举器来定义其方向：

|                   |                      |
| ----------------- | -------------------- |
| DMA_NONE          | 无方向（用于调试）   |
| DMA_TO_DEVICE     | 数据从内存传输到设备 |
| DMA_FROM_DEVICE   | 数据从设备传输到内存 |
| DMA_BIDIRECTIONAL | 方向未知             |

> [!NOTE]
>
> 并非机器中的所有内存区域都可以通过此 API 进行映射。此外，连续的内核虚拟空间可能在物理内存中不连续。由于此 API 不提供任何散列/聚集能力，如果用户尝试映射一个非物理连续的内存块，它将失败。因此，通过这个 API 映射的内存应该从可以保证其物理连续的来源获取（如 kmalloc）。
>
> 此外，内存的 DMA 地址必须在设备的 dma_mask 内（dma_mask 是设备可寻址区域的位掩码，即，如果内存的 DMA 地址与 dma_mask 进行与操作后仍然等于 DMA 地址，则设备可以对内存执行 DMA）。为了确保由 kmalloc 分配的内存位于 dma_mask 内，驱动程序可以指定各种依赖于平台的标志来限制分配的 DMA 地址范围（例如，在 x86上，GFP_DMA 保证在可用 DMA 地址的前 16MB 内，这是 ISA 设备所要求的）。
>
> 还要注意，如果平台有 IOMMU（一种将 I/O DMA 地址映射到物理内存地址的设备），上述有关物理连续性和 dma_mask 的限制可能不适用。然而，为了可移植性，设备驱动程序编写者不应假设存在这样的 IOMMU。

> [!WARNING]
>
> 内存一致性以称为缓存行宽度的粒度操作。为了使通过此 API 映射的内存正确运作，映射区域必须正好从一个缓存行边界开始并正好在一个缓存行边界结束（以防两个单独映射的区域共享一个缓存行）。由于编译时可能不知道缓存行大小，API 将不会强制执行此要求。因此，建议那些在运行时不特别注意确定缓存行大小的驱动程序编写者，只映射开始和结束在页面边界上的虚拟区域（这也保证是缓存行边界）。
>
> 在软件最后一次修改内存区域并在将其交给设备之前，必须进行 DMA_TO_DEVICE 同步。一旦使用了这个原语，此原语覆盖的内存就应该被设备视为只读。如果设备可能在任何时候写入它，则应使用 DMA_BIDIRECTIONAL（见下文）。
>
> 在驱动程序访问可能被设备更改的数据之前，必须进行 DMA_FROM_DEVICE 同步。此内存应被驱动程序视为只读。如果驱动程序需要在任何时候写入它，则应使用 DMA_BIDIRECTIONAL（见下文）。
>
> DMA_BIDIRECTIONAL 需要特殊处理：这意味着驱动程序不确定内存在交给设备之前是否被修改，也不确定设备是否也会修改它。因此，必须总是两次同步双向内存：一次是在内存交给设备之前（以确保所有内存更改都从处理器中刷新），一次是在设备使用后可能访问数据之前（以确保任何处理器缓存行都更新了设备可能更改的数据）。

```c
void dma_unmap_single(struct device *dev, dma_addr_t dma_addr, size_t size,
                 enum dma_data_direction direction)
```

取消先前区域的映射。所有传入的参数必须与通过映射 API 传入（并返回）的参数完全相同。

```c
dma_addr_t dma_map_page(struct device *dev, struct page *page,
             unsigned long offset, size_t size,
             enum dma_data_direction direction)

void dma_unmap_page(struct device *dev, dma_addr_t dma_address, size_t size,
             enum dma_data_direction direction)
```

用于映射和取消映射页面的 API 。其他映射 API 的所有注释和警告都适用于此处。另外，尽管提供了 \<offset\> 和 \<size\> 参数以进行部分页面映射，除非你真正了解缓存宽度，否则建议你永远不要使用这些参数。

```c
dma_addr_t dma_map_resource(struct device *dev, phys_addr_t phys_addr, size_t size,
                 enum dma_data_direction dir, unsigned long attrs)

void dma_unmap_resource(struct device *dev, dma_addr_t addr, size_t size,
                 enum dma_data_direction dir, unsigned long attrs)
```

用于映射和取消映射 MMIO 资源的API。其他映射 API 的所有注释和警告都适用于此处。API 只能用于映射设备 MMIO 资源，不允许映射 RAM 。

```c
int dma_mapping_error(struct device *dev, dma_addr_t dma_addr)
```

在某些情况下，dma_map_single()、dma_map_page() 和 dma_map_resource() 将无法创建映射。驱动程序可以通过使用 dma_mapping_error() 测试返回的 DMA 地址来检查这些错误。非 0 返回值意味着无法创建映射，驱动程序应采取适当的措施（例如，减少当前 DMA 映射的使用或延迟并稍后重试）。

```c
int dma_map_sg(struct device *dev, struct scatterlist *sg,
           int nents, enum dma_data_direction direction)
```

返回：映射的 DMA 地址段的数量（如果散列/聚集列表的某些元素在物理上或虚拟上相邻，并且 IOMMU 用单个条目映射它们，则返回值可能比传入的 \<nents\> 更短）。

请注意，如果 sg 已经被映射过一次，则不能再次映射 sg 。映射过程被允许破坏 sg 中的信息。

与其他映射接口一样，dma_map_sg() 也可能失败。当它失败时，返回 0，并且驱动程序必须采取适当的行动。对于驱动程序来说，采取一些行动是至关重要的，即使是块驱动程序中止请求或发生内核错误也比什么都不做好，因为什么都不做可能会损坏文件系统。

使用散列表时，您可以像这样使用由此产生的映射：

```c
int i, count = dma_map_sg(dev, sglist, nents, direction);
struct scatterlist *sg;

for_each_sg(sglist, sg, count, i) {
        hw_address[i] = sg_dma_address(sg);
        hw_len[i] = sg_dma_len(sg);
}
```

其中 nents 是 sglist 中的条目数量。

该实现可以自由地将几个连续的 sglist 条目合并为一个（例如，使用 IOMMU，或者如果几个页面恰好物理连续）并返回它映射到的实际 sg 条目数。失败时返回 0 。

然后您应该循环 count 次（注意：这可以少于 nents 次），并使用 sg_dma_address() 和 sg_dma_len() 宏来获取DMA映射的地址和长度，代替之前直接访问散列表元素的 sg->address 和 sg->length 属性。

```c
void dma_unmap_sg(struct device *dev, struct scatterlist *sg,
             int nents, enum dma_data_direction direction)
```

取消先前散列/聚集列表的映射。所有参数必须与传入散列/聚集映射 API 的参数相同。

注意：\<nents\> 必须是您传入的数量，而不是返回的 DMA 地址条目的数量。

```c
void dma_sync_single_for_cpu(struct device *dev, dma_addr_t dma_handle,
                        size_t size, enum dma_data_direction direction)

void dma_sync_single_for_device(struct device *dev, dma_addr_t dma_handle,
                        size_t size, enum dma_data_direction direction)

void dma_sync_sg_for_cpu(struct device *dev, struct scatterlist *sg,
                        int nents, enum dma_data_direction direction)

void dma_sync_sg_for_device(struct device *dev, struct scatterlist *sg,
                        int nents, enum dma_data_direction direction)
```

为 CPU 和设备同步单个连续映射或散列/聚集映射。使用 sync_sg API，所有参数必须与传入 sg 映射 API 的参数相同。使用 sync_single API，可以使用与传入单个映射 API 的参数不完全相同的 dma_handle 和 size 参数来进行部分同步。

> [!NOTE]
>
> 你必须这样做：
>
> - 在从设备读取由 DMA 写入的值之前（使用 DMA_FROM_DEVICE 方向）
> - 在写入将通过 DMA 写入设备的值之后（使用 DMA_TO_DEVICE 方向）
> - 在将内存交给设备的之前和之后，如果内存是 DMA_BIDIRECTIONAL

> 译者注：即进行 CPU 和设备同步的三个时间节点。

另请参见 dma_map_single() 。

```c
dma_addr_t dma_map_single_attrs(struct device *dev, void *cpu_addr, size_t size,
                     enum dma_data_direction dir,
                     unsigned long attrs)

void dma_unmap_single_attrs(struct device *dev, dma_addr_t dma_addr,
                       size_t size, enum dma_data_direction dir,
                       unsigned long attrs)

int dma_map_sg_attrs(struct device *dev, struct scatterlist *sgl,
                 int nents, enum dma_data_direction dir,
                 unsigned long attrs)

void dma_unmap_sg_attrs(struct device *dev, struct scatterlist *sgl,
                   int nents, enum dma_data_direction dir,
                   unsigned long attrs)
```

上述四个函数与没有 _attrs 后缀的对应函数类似，它们只是传递了一个可选的 dma_attrs 。

DMA 属性的解释是特定于体系结构的，每个属性应在 [DMA 属性](https://docs.kernel.org/core-api/dma-attributes.html)中被记录。

如果 dma_attrs 为 0，则这些函数的语义与没有  _attrs 后缀的相应函数相同。因此，dma_map_single_attrs() 通常可以替代 dma_map_single()，其他函数同理。

作为使用 *_attrs 函数的一个例子，这里展示了如何在为 DMA 映射内存时传递一个属性 DMA_ATTR_FOO：

```c
#include <linux/dma-mapping.h>
/* DMA_ATTR_FOO should be defined in linux/dma-mapping.h and
* documented in Documentation/core-api/dma-attributes.rst */
...

        unsigned long attr;
        attr |= DMA_ATTR_FOO;
        ....
        n = dma_map_sg_attrs(dev, sg, nents, DMA_TO_DEVICE, attr);
        ....
```

关心 DMA_ATTR_FOO 的体系结构会在它们的映射和取消映射函数的实现中检查它的存在，例如：

```c
void whizco_dma_map_sg_attrs(struct device *dev, dma_addr_t dma_addr,
                             size_t size, enum dma_data_direction dir,
                             unsigned long attrs)
{
        ....
        if (attrs & DMA_ATTR_FOO)
                /* twizzle the frobnozzle */
        ....
}
```

## 第二部分 —— 非一致性 DMA 分配

这些 API 允许分配保证能被传入设备 DMA 寻址的页面，但需要对内核与设备的内存所有权进行显式管理。

如果你不了解缓存行一致性在处理器和 I/O 设备之间的工作原理，你不应该使用这部分 API。

```c
struct page *dma_alloc_pages(struct device *dev, size_t size, dma_addr_t *dma_handle,
                enum dma_data_direction dir, gfp_t gfp)
```

此函数分配一个 <size\> 字节的非一致性内存区域。它返回指向该区域第一个页面结构体的指针，如果分配失败则返回 NULL 。生成的页面结构体可以用于一切适合页面结构体的用途。

> 译者注：页面结构体即 struct page 。

它还返回一个 \<dma_handle\> ，它可以被转换为与总线宽度相同的无符号整数，并作为该区域的 DMA 基地址提供给设备。

dir 参数指定设备是读取还是写入数据，详细信息请参见 dma_map_single() 。

gfp 参数允许调用者指定分配的 GFP_ 标志（参见 [`kmalloc()`](https://docs.kernel.org/core-api/mm-api.html#c.kmalloc)），但拒绝用于指定内存区域的标志，如 GFP_DMA 或 GFP_HIGHMEM 。

在将内存交给设备之前，需要调用 dma_sync_single_for_device()，在读取设备写入的内存之前，需要调用 dma_sync_single_for_cpu() ，就像重复使用的流式 DMA 映射一样。

```c
void dma_free_pages(struct device *dev, size_t size, struct page *page,
                dma_addr_t dma_handle, enum dma_data_direction dir)
```

释放之前使用 dma_alloc_pages() 分配的内存区域。dev、size、dma_handle 和 dir 必须与传递给dma_alloc_pages() 的相同。page 必须是 dma_alloc_pages() 返回的指针。

```c
int dma_mmap_pages(struct device *dev, struct vm_area_struct *vma,
               size_t size, struct page *page)
```

将 dma_alloc_pages() 返回的分配映射到用户地址空间。dev 和 size 必须与传入 dma_alloc_pages() 的相同。page 必须是 dma_alloc_pages() 返回的指针。

```c
void *dma_alloc_noncoherent(struct device *dev, size_t size,
                dma_addr_t *dma_handle, enum dma_data_direction dir,
                gfp_t gfp)
```

这个函数是 dma_alloc_pages 的便利包装器，它返回分配内存的内核虚拟地址，而不是页面结构体。

```c
void dma_free_noncoherent(struct device *dev, size_t size, void *cpu_addr,
                dma_addr_t dma_handle, enum dma_data_direction dir)
```

释放之前使用 dma_alloc_noncoherent() 分配的内存区域。dev、size、dma_handle 和 dir 必须与传入dma_alloc_noncoherent() 的相同。cpu_addr 必须是 dma_alloc_noncoherent() 返回的虚拟地址。

```c
struct sg_table *dma_alloc_noncontiguous(struct device *dev, size_t size,
                        enum dma_data_direction dir, gfp_t gfp,
                        unsigned long attrs);
```

此函数分配 \<size\> 字节的非一致性且可能是非连续的内存。它返回一个指向 sg_table 结构体的指针，该指针描述已分配和 DMA 映射的内存，如果分配失败则返回 NULL 。得到的内存可用于映射到散列表中的页结构体。

返回的 sg_table 保证有一个单独的 DMA 映射段，如 sgt->nents 所示，但它可能有多个 CPU 的段，如 sgt->orig_nents 所示。

dir 参数指定设备是读取还是写入数据，详细信息请参见 dma_map_single() 。

gfp 参数允许调用者指定分配的 GFP_ 标志（参见 [`kmalloc()`](https://docs.kernel.org/core-api/mm-api.html#c.kmalloc)），但拒绝用于指定内存区域的标志，如 GFP_DMA 或 GFP_HIGHMEM 。

attrs 参数必须是 0 或 DMA_ATTR_ALLOC_SINGLE_PAGES 。

在将内存交给设备之前，需要调用 dma_sync_sgtable_for_device()，在读取设备写入的内存之前，需要调用 dma_sync_sgtable_for_cpu() ，就像重复使用的流式 DMA 映射一样。

```c
void dma_free_noncontiguous(struct device *dev, size_t size,
                       struct sg_table *sgt,
                       enum dma_data_direction dir)
```

释放之前使用 dma_alloc_noncontiguous() 分配的内存。dev、size 和 dir 必须与传入dma_alloc_noncontiguous() 的相同。sgt 必须是 dma_alloc_noncontiguous() 返回的指针。

```c
void *dma_vmap_noncontiguous(struct device *dev, size_t size,
        struct sg_table *sgt)
```

为 dma_alloc_noncontiguous() 返回的分配返回一个连续的内核映射。dev 和 size 必须与传入 dma_alloc_noncontiguous() 的相同。sgt 必须是 dma_alloc_noncontiguous() 返回的指针。

一旦使用此函数映射了一个非连续分配，就必须使用 flush_kernel_vmap_range() 和 invalidate_kernel_vmap_range() API 来管理内核映射、设备和用户空间映射（如果有）之间的一致性。

```c
void dma_vunmap_noncontiguous(struct device *dev, void *vaddr)
```

取消映射由 dma_vmap_noncontiguous() 返回的内核映射。dev 必须与传入 dma_alloc_noncontiguous() 的相同。vaddr 必须是 dma_vmap_noncontiguous() 返回的指针。

```c
int dma_mmap_noncontiguous(struct device *dev, struct vm_area_struct *vma,
                       size_t size, struct sg_table *sgt)
```

将从 dma_alloc_noncontiguous() 返回的分配映射到用户地址空间。dev 和 size 必须与传入 dma_alloc_noncontiguous() 的相同。sgt 必须是 dma_alloc_noncontiguous() 返回的指针。

```c
int dma_get_cache_alignment(void)
```

返回处理器缓存对齐方式。这是在映射内存或进行部分刷新时必须遵守的绝对最小对齐和宽度。

> [!NOTE]
>
> 这个 API 可能返回一个大于实际缓存行的数字，但它将保证一个或多个缓存行正好适合这个调用返回的宽度。为了便于对齐，它也始终是 2 的幂。

## 第三部分 —— 调试驱动程序对 DMA-API 的使用

上文介绍的 DMA-API 有一些限制。例如，DMA 地址必须使用相同大小的相应函数释放。随着硬件 IOMMU 的出现，驱动程序不违反这些约束变得越来越重要。在最坏的情况下，这种违规行为可能会导致数据损坏，甚至破坏文件系统。

为了调试驱动程序并发现 DMA-API 使用中的错误，可以将检查代码编译到内核中，这将告诉开发者这些违规行为。如果您的体系结构支持，您可以在内核配置中选择 “Enable debugging of DMA-API usage” 选项。启用这个选项会影响性能。不要在生产内核中启用它。

如果启用，生成的内核将包含一些代码，这些代码会对哪个设备分配了哪些 DMA 内存进行一些记录。如果这段代码检测到错误，它会打印一条带有一些详细信息的警告消息到您的内核日志中。一个警告消息示例可能如下所示：

```
WARNING: at /data2/repos/linux-2.6-iommu/lib/dma-debug.c:448
        check_unmap+0x203/0x490()
Hardware name:
forcedeth 0000:00:08.0: DMA-API: device driver frees DMA memory with wrong
        function [device address=0x00000000640444be] [size=66 bytes] [mapped as
single] [unmapped as page]
Modules linked in: nfsd exportfs bridge stp llc r8169
Pid: 0, comm: swapper Tainted: G        W  2.6.28-dmatest-09289-g8bb99c0 #1
Call Trace:
<IRQ>  [<ffffffff80240b22>] warn_slowpath+0xf2/0x130
[<ffffffff80647b70>] _spin_unlock+0x10/0x30
[<ffffffff80537e75>] usb_hcd_link_urb_to_ep+0x75/0xc0
[<ffffffff80647c22>] _spin_unlock_irqrestore+0x12/0x40
[<ffffffff8055347f>] ohci_urb_enqueue+0x19f/0x7c0
[<ffffffff80252f96>] queue_work+0x56/0x60
[<ffffffff80237e10>] enqueue_task_fair+0x20/0x50
[<ffffffff80539279>] usb_hcd_submit_urb+0x379/0xbc0
[<ffffffff803b78c3>] cpumask_next_and+0x23/0x40
[<ffffffff80235177>] find_busiest_group+0x207/0x8a0
[<ffffffff8064784f>] _spin_lock_irqsave+0x1f/0x50
[<ffffffff803c7ea3>] check_unmap+0x203/0x490
[<ffffffff803c8259>] debug_dma_unmap_page+0x49/0x50
[<ffffffff80485f26>] nv_tx_done_optimized+0xc6/0x2c0
[<ffffffff80486c13>] nv_nic_irq_optimized+0x73/0x2b0
[<ffffffff8026df84>] handle_IRQ_event+0x34/0x70
[<ffffffff8026ffe9>] handle_edge_irq+0xc9/0x150
[<ffffffff8020e3ab>] do_IRQ+0xcb/0x1c0
[<ffffffff8020c093>] ret_from_intr+0x0/0xa
<EOI> <4>---[ end trace f6435a98e2a38c0e ]---
```

驱动程序开发者可以找到驱动程序和设备，包括导致此警告的 DMA-API 调用的堆栈跟踪。

默认情况下，只有第一个错误才会导致警告消息。所有其他错误只会被静默计数。这个限制存在是为了防止代码淹没您的内核日志。为了支持调试设备驱动程序，可以通过 debugfs 禁用此功能。有关详细信息，请参阅下面的 debugfs 接口文档。

DMA-API 调试代码的 debugfs 目录称为 dma-api/ 。在这个目录中，当前可以找到以下文件：

|                          |                                                                                                                                        |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| dma-api/all_errors       | 此文件包含一个数值。如果这个值不等于零，调试代码将在内核日志中打印它发现的每个错误的警告。小心使用此选项，因为它可以很容易地淹没日志。 |
| dma-api/disabled         | 如果调试代码被禁用，则此只读文件包含字符“Y”。这可能发生在它用完内存或在启动时被禁用。                                                  |
| dma-api/dump             | 此只读文件包含当前的 DMA 映射。                                                                                                        |
| dma-api/error_count      | 此文件是只读的，显示发现的错误总数。                                                                                                   |
| dma-api/num_errors       | 此文件中的数字显示在停止之前将打印多少警告到内核日志。系统启动时，这个数字被初始化为 1，并且可以通过写入此文件来设置。                 |
| dma-api/min_free_entries | 可以读取此只读文件以获得分配器所见过的最小可用 dma_debug_entries 数。如果该值降至零，代码将尝试增加 nr_total_entries 以进行补偿。      |
| dma-api/num_free_entries | 当前分配器中剩余的 dma_debug_entries 数量。                                                                                            |
| dma-api/nr_total_entries | 分配器中的总 dma_debug_entries 数量，包括空闲和已用的。                                                                                |
| dma-api/driver_filter    | 您可以将驱动程序的名称写入此文件，以将调试输出限制为该特定驱动程序的请求。向该文件写入空字符串以禁用过滤器并再次查看所有错误。         |

如果将此代码编译到内核中，默认情况下它将被启用。如果您想在不记账的情况下启动，您可以提供 “dma_debug=off” 作为启动参数。这将禁用 DMA-API 调试。注意，您不能在运行时再次启用它。您必须重新启动才能这样做。

如果您只想看到特殊设备驱动程序的调试消息，您可以指定 dma_debug_driver=\<drivername\> 参数。这将在启动时启用驱动程序过滤器。此后，调试代码将只打印该驱动程序的错误。这个过滤器可以稍后使用 debugfs 禁用或更改。

当代码在运行时自行禁用时，这最有可能是因为它用尽了 dma_debug_entries 并且无法根据需求分配更多。启动时预先分配了 65536 个条目——如果这对您来说太低，用 “dma_debug_entries=<your_desired_number>” 启动以覆盖默认值。注意，代码会批量分配条目，所以实际预先分配的条目数量可能大于实际请求的数量。每当代码动态分配了与最初预分配一样多的条目时，它都会打印到内核日志中。这表明更大的预分配大小可能是合适的，或者如果它持续发生，表明驱动程序可能存在泄漏映射的情况。

```c
void debug_dma_mapping_error(struct device *dev, dma_addr_t dma_addr);
```

dma-debug 接口 debug_dma_mapping_error() 用于调试无法检查 dma_map_single() 和 dma_map_page() 接口返回的地址上的 DMA 映射错误的驱动程序。此接口清除由 debug_dma_map_page() 设置的标志，以表明驱动程序已调用 dma_mapping_error()。当驱动程序取消映射时，debug_dma_unmap() 会检查该标志，如果该标志仍处于设置状态，则会打印警告消息，其中包括导致取消映射的调用跟踪。可以从 dma_mapping_error() 例程调用此接口，以启用 DMA 映射错误检查调试。
