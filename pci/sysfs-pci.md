> 原文：[5. Accessing PCI device resources through sysfs — The Linux Kernel documentation](https://docs.kernel.org/PCI/sysfs-pci.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.22
>
> 修订：

# 5. 通过 sysfs 访问 PCI 设备资源

sysfs 通常挂载在 /sys 上，提供对支持它的平台上 PCI 资源的访问。例如，给定的总线可能如下所示：

```
/sys/devices/pci0000:17
|-- 0000:17:00.0
|   |-- class
|   |-- config
|   |-- device
|   |-- enable
|   |-- irq
|   |-- local_cpus
|   |-- remove
|   |-- resource
|   |-- resource0
|   |-- resource1
|   |-- resource2
|   |-- revision
|   |-- rom
|   |-- subsystem_device
|   |-- subsystem_vendor
|   `-- vendor
`-- ...
```

最顶层的元素描述了 PCI 域和总线号。在这个例子中，域号为 0000，总线号为 17（两个值都是十六进制）。这个总线在插槽 0 中包含一个单一功能设备。为了方便起见，重复显示了域号和总线号。在设备目录下有几个文件，每个文件都有其自己的功能。

| 文件               | 功能                                                         |
| ------------------ | ------------------------------------------------------------ |
| class              | PCI 类（ASCII 文本，只读）                                   |
| config             | PCI 配置（二进制数据，读写）                                 |
| device             | PCI 设备（ASCII 文本，只读）                                 |
| enable             | 设备是否启用（ASCII 文本，读写）                             |
| irq                | IRQ 编号（ASCII 文本，只读）                                 |
| local_cpus         | 附近的 CPU 掩码（CPU 掩码，只读）                            |
| remove             | 从内核列表中删除设备（ASCII 文本，只读）                     |
| resource           | PCI 资源主机地址（ASCII 文本，只读）                         |
| resource0..N       | PCI 资源 N，如果存在（二进制数据，可以通过 mmap 映射，仅 IORESOURCE_IO 区域的读写） |
| resource0_wc..N_wc | PCI 的写组合映射资源 N，如果可预取（二进制数据，可以使用 mmap） |
| revision           | PCI 修订号（ASCII 文本，只读）                               |
| rom                | PCI ROM 资源，如果存在（二进制数据，只读）                   |
| subsystem_device   | PCI 子系统设备（ASCII 文本，只读）                           |
| subsystem_vendor   | PCI 子系统供应商（ASCII 文本，只读）                         |
| vendor             | PCI 供应商（ASCII 文本，只读）                               |

只读文件是用于提供信息的，对它们的写操作将被忽略（“rom” 文件除外）。可写文件可以用来对设备执行操作（例如，改变配置空间，卸载设备）。可通过 mmap 映射的文件可通过偏移量为 0 的文件 mmap 获得，可用于从用户空间进行实际的设备编程。注意，某些平台不支持某些资源的内存映射，因此在尝试进行 mmap 操作时一定要检查返回值。最值得注意的是 I/O 端口资源，它们同样提供读/写访问权限。

“enable” 文件提供一个计数器，指示设备被启用的次数。如果 “enable” 文件当前返回 “4”，并且向其 echo “1”，那么它将返回 “5” 。echo “0” 会减少计数。即使计数返回到 0，一些初始化操作也可能不会被逆转。

“rom” 文件的特殊之处在于，它提供了对设备 ROM 文件的只读访问（如果可用）。但它默认是禁用的，因此应用程序应该在尝试读取操作之前，先写入字符串 "1" 以启用它，并在访问后通过写入 "0" 来禁用它。请注意，必须启用设备才能成功地从 rom 读取数据。如果驱动程序未绑定到设备，则可以使用上面介绍的 “enable” 文件来启用它。

“remove” 文件用于通过写入一个非零整数来移除 PCI 设备。这不涉及任何类型的热插拔功能，例如关闭设备电源。设备将从内核的 PCI 设备列表中移除，其 sysfs 目录将被删除，并且设备将从任何连接到它的驱动程序中移除。不允许移除 PCI 根总线。

## 5.1. 通过 sysfs 访问传统资源

如果底层平台支持，传统的 I/O 端口和 ISA 内存资源也会在 sysfs 中提供。它们位于 PCI 类层次结构中，例如：

```
/sys/class/pci_bus/0000:17/
|-- bridge -> ../../../devices/pci0000:17
|-- cpuaffinity
|-- legacy_io
`-- legacy_mem
```

legacy_io 文件是一个可读写的文件，应用程序可以使用它来执行传统端口 I/O 。应用程序应该打开文件，查找所需端口（例如 0x3e8），然后执行 1、2或4 字节的读或写操作。legacy_mem 文件应该使用与所需内存偏移相对应的偏移进行内存映射，例如，使用 0xa0000 进行 VGA 帧缓冲区的映射。应用程序然后可以简单地解引用返回的指针（当然要先检查错误）来访问传统内存空间。

## 5.2. 支持新平台上的 PCI 访问

为了支持如上所述的 PCI 资源映射，Linux 平台代码理想情况下应该定义 ARCH_GENERIC_PCI_MMAP_RESOURCE 并使用该功能的通用实现。为了支持通过 /proc/bus/pci 中的文件进行 mmap() 的历史接口，平台也可以设置 HAVE_PCI_MMAP 。

另一方面，设置了 HAVE_PCI_MMAP 的平台可以提供自己的 pci_mmap_resource_range() 实现，而不是定义 ARCH_GENERIC_PCI_MMAP_RESOURCE 。

支持 PCI 资源写组合映射的平台必须定义 arch_can_pci_mmap_wc() ，在允许写组合时，该函数在运行时应评估为非零。同样，支持 I/O 资源映射的平台定义 arch_can_pci_mmap_io()。

传统资源受 HAVE_PCI_LEGACY 定义的保护。希望支持传统功能的平台应该定义它，并提供 pci_legacy_read 、 pci_legacy_write 和 pci_mmap_legacy_page_range 函数。
