> 原文：[Core API Documentation — The Linux Kernel documentation](https://docs.kernel.org/core-api/index.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.20
>
> 修订：
>
> **已知问题：二级目录需要右键-在新标签页中打开链接才能正确跳转**

# 核心 API 文档

这是核心内核 API 手册的开始。非常感谢为本手册改变（和编写！）文档！

## 核心工具

This section has general and "core core" documentation. The first is a massive grab-bag of kerneldoc info left over from the docbook days; it should really be broken up someday when somebody finds the energy to do it.

- [The Linux Kernel API](https://docs.kernel.org/core-api/kernel-api.html)
- [Workqueue](https://docs.kernel.org/core-api/workqueue.html)
- [General notification mechanism](https://docs.kernel.org/core-api/watch_queue.html)
- [Message logging with printk](https://docs.kernel.org/core-api/printk-basics.html)
- [How to get printk format specifiers right](https://docs.kernel.org/core-api/printk-formats.html)
- [Printk Index](https://docs.kernel.org/core-api/printk-index.html)
- [Symbol Namespaces](https://docs.kernel.org/core-api/symbol-namespaces.html)
- [Assembler Annotations](https://docs.kernel.org/core-api/asm-annotations.html)

## Data structures and low-level utilities

Library functionality that is used throughout the kernel.

- [Everything you never wanted to know about kobjects, ksets, and ktypes](https://docs.kernel.org/core-api/kobject.html)
- [Adding reference counters (krefs) to kernel objects](https://docs.kernel.org/core-api/kref.html)
- [Generic Associative Array Implementation](https://docs.kernel.org/core-api/assoc_array.html)
- [XArray](https://docs.kernel.org/core-api/xarray.html)
- [Maple Tree](https://docs.kernel.org/core-api/maple_tree.html)
- [ID Allocation](https://docs.kernel.org/core-api/idr.html)
- [Circular Buffers](https://docs.kernel.org/core-api/circular-buffers.html)
- [Red-black Trees (rbtree) in Linux](https://docs.kernel.org/core-api/rbtree.html)
- [Generic radix trees/sparse arrays](https://docs.kernel.org/core-api/generic-radix-tree.html)
- [Generic bitfield packing and unpacking functions](https://docs.kernel.org/core-api/packing.html)
- [this_cpu operations](https://docs.kernel.org/core-api/this_cpu_ops.html)
- [ktime accessors](https://docs.kernel.org/core-api/timekeeping.html)
- [The errseq_t datatype](https://docs.kernel.org/core-api/errseq.html)
- [Atomic types](https://docs.kernel.org/core-api/wrappers/atomic_t.html)
- [Atomic bitops](https://docs.kernel.org/core-api/wrappers/atomic_bitops.html)

## Low level entry and exit

- [Entry/exit handling for exceptions, interrupts, syscalls and KVM](https://docs.kernel.org/core-api/entry.html)

## Concurrency primitives

How Linux keeps everything from happening at the same time. See [Locking](https://docs.kernel.org/locking/index.html) for more related documentation.

- [refcount_t API compared to atomic_t](https://docs.kernel.org/core-api/refcount-vs-atomic.html)
- [IRQs](https://docs.kernel.org/core-api/irq/index.html)
- [Semantics and Behavior of Local Atomic Operations](https://docs.kernel.org/core-api/local_ops.html)
- [The padata parallel execution mechanism](https://docs.kernel.org/core-api/padata.html)
- [RCU concepts](https://docs.kernel.org/RCU/index.html)
- [Linux kernel memory barriers](https://docs.kernel.org/core-api/wrappers/memory-barriers.html)

## Low-level hardware management

Cache management, managing CPU hotplug, etc.

- [Cache and TLB Flushing Under Linux](https://docs.kernel.org/core-api/cachetlb.html)
- [CPU hotplug in the Kernel](https://docs.kernel.org/core-api/cpu_hotplug.html)
- [Memory hotplug](https://docs.kernel.org/core-api/memory-hotplug.html)
- [Linux generic IRQ handling](https://docs.kernel.org/core-api/genericirq.html)
- [Memory Protection Keys](https://docs.kernel.org/core-api/protection-keys.html)

## Memory management

如何在内核中分配和使用内存。请注意，在[内存管理文档](https://docs.kernel.org/mm/index.html)中有更多的相关文档。

- [Memory Allocation Guide](https://docs.kernel.org/core-api/memory-allocation.html)
- [Unaligned Memory Accesses](https://docs.kernel.org/core-api/unaligned-memory-access.html)
- [使用通用设备的动态 DMA 映射](dma-api.md)
- [Dynamic DMA mapping Guide](https://docs.kernel.org/core-api/dma-api-howto.html)
- [DMA attributes](https://docs.kernel.org/core-api/dma-attributes.html)
- [DMA with ISA and LPC devices](https://docs.kernel.org/core-api/dma-isa-lpc.html)
- [Memory Management APIs](https://docs.kernel.org/core-api/mm-api.html)
- [The genalloc/genpool subsystem](https://docs.kernel.org/core-api/genalloc.html)
- [pin_user_pages() and related calls](https://docs.kernel.org/core-api/pin_user_pages.html)
- [Boot time memory management](https://docs.kernel.org/core-api/boot-time-mm.html)
- [GFP masks used from FS/IO context](https://docs.kernel.org/core-api/gfp_mask-from-fs-io.html)

## Interfaces for kernel debugging

- [The object-lifetime debugging infrastructure](https://docs.kernel.org/core-api/debug-objects.html)
- [The Linux Kernel Tracepoint API](https://docs.kernel.org/core-api/tracepoint.html)
- [Using physical DMA provided by OHCI-1394 FireWire controllers for debugging](https://docs.kernel.org/core-api/debugging-via-ohci1394.html)

## Everything else

Documents that don't fit elsewhere or which have yet to be categorized.

- [Reed-Solomon Library Programming Interface](https://docs.kernel.org/core-api/librs.html)
- [Netlink notes for kernel developers](https://docs.kernel.org/core-api/netlink.html)
