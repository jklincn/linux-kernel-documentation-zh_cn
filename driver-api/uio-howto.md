> 原文：[The Userspace I/O HOWTO — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/driver-api/uio-howto.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.04.20
>
> 修订：

# 用户空间 I/O 指南

## 关于这个文档

### 翻译

如果您知道这份文件的任何翻译，或者您有兴趣翻译它，请给我 hjk@hansjkoch.de 发电子邮件。

### 前言

对于许多类型的设备而言，创建一个 Linux 内核驱动是大材小用的。真正需要的只是某种方式来处理中断，并提供对设备内存空间的访问。控制设备的逻辑不一定必须在内核中，因为设备不需要利用内核提供的其他资源。像这样的一类常见设备为工业 I/O 卡。

为了解决这种情况，用户空间 I/O 系统（UIO）被设计出来。对于典型的工业 I/O 卡，只需要一个非常小的内核模块。驱动程序的主要部分将在用户空间中运行。这简化了开发过程，并降低了内核模块中出现严重错误的风险。

请注意，UIO 不是一个通用的驱动程序接口。那些已经由其他内核子系统（如网络、串行或 USB）很好地处理的设备，不适用于 UIO 驱动。适用于 UIO 驱动的硬件应满足以下所有条件：

- 设备具有可以映射的内存。设备可以通过写入这些内存来完全控制。
- 设备通常会产生中断。
- 设备不适合任何一个标准内核子系统。

### 致谢 

我要感谢 Linutronix 的 Thomas Gleixner 和 Benedikt Spranger，他们不仅编写了大部分 UIO 代码，还通过提供各种背景信息，极大地帮助了我编写这个指南。

### 反馈

发现本文档有错误吗？（或者也许是对的？）我很希望收到您的反馈。请通过 hjk@hansjkoch.de 给我发送电子邮件。

## 关于 UIO

如果您使用 UIO 作为卡的驱动程序，您将获得：

- 只需编写和维护一个小的内核模块。
- 使用您习惯的所有工具和库，在用户空间开发驱动程序的主要部分。
- 驱动程序中的错误不会导致内核崩溃。
- 驱动程序的更新可以在不重新编译内核的情况下进行。

### UIO 工作原理

每个 UIO 设备都通过一个设备文件和多个 sysfs 属性文件进行访问。第一个设备的设备文件将被命名为 `/dev/uio0`，随后的设备依次为 `/dev/uio1`、`/dev/uio2` 等。

`/dev/uioX` 用于访问卡的地址空间。只需使用 **mmap()** 访问卡的寄存器或 RAM 位置即可。

中断是通过从 `/dev/uioX` 中读取来处理的。一旦发生中断，`/dev/uioX` 的阻塞 **read()** 将立即返回。您也可以在 `/dev/uioX` 上使用 **select()** 来等待中断。从 `/dev/uioX` 读取的整数值表示总中断计数。您可以使用这个数字来判断您是否错过了一些中断。

对于某些内部有多个中断源，但没有单独的 IRQ 掩码和状态寄存器的硬件，如果内核处理程序通过写入芯片的 IRQ 寄存器来禁用中断源，则用户空间可能无法确定中断源是什么。在这种情况下，内核必须完全禁用 IRQ 才能使芯片的寄存器保持不变。现在用户空间部分可以确定中断的原因，但它不能重新启用中断。另一个特例是重新启用中断是对合并的 IRQ 状态/确认寄存器的读取-修改-写入操作的芯片。如果同时发生一个新的中断，这将是不稳定的。

为了解决这些问题，UIO 还实现了一个 write() 函数。对于只有单个中断源或具有独立 IRQ 掩码和状态寄存器的硬件，通常不使用它，也可以忽略它。但是，如果您需要它，对 `/dev/uioX` 的写入将调用驱动程序实现的 **irqcontrol()** 函数。您必须写入一个 32 位值，该值通常为 0 或 1 才能禁用或启用中断。如果驱动程序没有实现 **irqcontrol()**，**write()** 将返回 `-ENOSYS` 。

为了正确处理中断，您的自定义内核模块可以提供自己的中断处理程序。它将自动由内置处理程序调用。

对于不生成中断但需要轮询的卡，可以设置一个计时器，以可配置的时间间隔触发中断处理程序。此中断模拟是通过从计时器的事件处理程序调用 **uio_event_notify()** 来完成的。

每个驱动程序都提供用于读取或写入变量的属性。这些属性可以通过 sysfs 文件来访问。自定义内核驱动程序模块可以将自己的属性添加到 uio 驱动程序所拥有的设备中，但此时不能添加到 UIO 设备本身。如果发现它有用的话，这种情况在将来可能会发生改变。

UIO 框架提供了以下标准属性：

- `name`：您的设备的名称。对此建议使用您的内核模块的名称。
- `version`：由驱动程序定义的版本字符串。这允许驱动程序的用户空间部分处理不同版本的内核模块。
- `event`：自上次读取设备节点以来，驱动程序处理的中断总数。

这些属性显示在 `/sys/class/uio/uioX` 目录下。请注意，此目录可能是符号链接，而不是真正的目录。任何访问它的用户空间代码都必须能够处理此问题。

每个 UIO 设备可以使一个或多个内存区域用于内存映射。这是必要的，因为一些工业 I/O 卡需要在驱动程序中访问多个 PCI 内存区域。

每个映射在 sysfs 中都有自己的目录，第一个映射显示为 `/sys/class/uio/uioX/maps/map0/` 。随后的映射会创建目录`map1/`，`map2/` 等。只有当映射的大小不为 0 时，这些目录才会出现。

每个 `mapX/` 目录包含 4 个只读文件，这些文件显示内存的属性：

- `name`：此映射的字符串标识符。这是可选的，字符串可以为空。驱动程序可以设置此项，以便用户空间更容易地找到正确的映射。
- `addr`：可以映射的内存地址。
- `size`：addr 指向的内存的大小（以字节为单位）。
- `offset`：偏移量，以字节为单位，必须添加到 **mmap()** 返回的指针中才能获得实际的设备内存。如果设备的内存没有页面对齐，则这一点会非常重要。请记住，**mmap()** 返回的指针总是页面对齐的，所以总是添加这个偏移量是一种很好的风格。

在用户空间中，通过调整 **mmap()** 调用的 `offset` 参数来区分不同的映射。要映射第 N 个内存区域，您必须使用页面大小的 N 倍作为偏移量：

```
offset = N * getpagesize();
```

有时，有些硬件具有类似内存的区域，无法使用此处描述的技术进行映射，但仍有方法从用户空间访问这些区域。最常见的例子是 x86 的 IO 端口。在 x86 系统上，用户空间可以使用 **ioperm()**，**iopl()**，**inb()**，**outb()** 和类似的函数访问这些 IO 端口。

由于这些 IO 端口区域无法映射，因此它们将不会像上述正常内存那样出现在 `/sys/class/uio/uioX/maps/` 下。如果没有关于这些端口区域的硬件提供的信息，驱动程序的用户空间部分就很难找出哪些端口属于哪个 UIO 设备。

为了解决这种情况，添加了新的目录 `/sys/class/uio/uioX/portio/` 。只有当驱动程序想将有关一个或多个端口区域的信息传递给用户空间时，它才存在。如果是这种情况，则名为 `port0`，`port1` 等的子目录将出现在 `/sys/class/uio/uioX/portio/` 下面。

每个 `portX/` 目录包含 4 个只读文件，表示端口区域的名称、开始、大小和类型：

- `name`：此端口区域的字符串标识符。该字符串是可选的，可以为空。驱动程序可以设置它，以便用户空间更容易找到某个端口区域。
- `start`：此区域的第一个端口。
- `size`：此区域中的端口数。
- `porttype`：描述端口类型的字符串。

## 编写您自己的内核模块

请看一下 `uio_cif.c` 作为一个例子。以下段落解释了本文件的不同部分。

### uio_info 结构体

这个结构体将驱动程序的详细信息告诉框架，一些成员是必需的，其他成员是可选的。

- `const char *name`：必需的。将显示在 sysfs 中的驱动程序的名称。建议使用您的模块名称。
- `const char *version`：必需的。此字符串出现在 `/sys/class/uio/uioX/version` 中。
- `struct-uio_mem mem[MAX_uio_MAPS]`：如果您有可以用 `mmap()` 映射的内存，则为必需项。对于每个映射，您需要填充一个 `uio_mem` 结构体。有关详细信息，请参阅下面的说明。
- `struct uio_port port[MAX_uio_PORTS_REGIONS]`：如果您要将 IO 端口的信息传递给用户空间，则为必需项。对于每个端口区域，您需要填充一个 `uio_port` 结构。有关详细信息，请参阅下面的说明。
- `long irq`：必需的。如果硬件产生中断，确定 IRQ 编号是模块在初始化期间的任务。如果您没有硬件生成的中断，但希望以其他方式触发中断处理程序，请将 irq 设置为 UIO_irq_CUSTOM 。如果您根本没有中断，您可以将 irq 设置为 UIO_irq_NONE，尽管这几乎没有意义。
- `unsigned long irq_flags`：如果您已将 irq 设置为硬件中断编号，则为必需项。这里给出的标志将用于对 **request_irq()** 的调用。
- `int (*mmap)(struct uio_info *info, struct vm_area_struct *vma)`：可选的。如果您需要一个特殊的 **mmap()** 函数，您可以在这里设置它。如果该指针不为 NULL，则将调用您的 **mmap()**，而不是内置的。
- `int (*open)(struct uio_info *info, struct inode *inode)`：可选的。您可能想要拥有自己的 **open()**，例如，仅当您的设备实际使用时才启用中断。
- `int (*release)(struct uio_info *info, struct inode *inode)`：可选的。如果您定义了自己的 **open()**，您可能还需要一个自定义的 **release()** 函数。
- `int (*irqcontrol)(struct uio_info *info, s32 irq_on)`：可选的。如果您需要能够通过写入 `/dev/uioX` 来启用或禁用来自用户空间的中断，则可以实现此功能。参数 `irq_on` 为 0 则禁用中断，为 1 则启用中断。

通常，您的设备会有一个或多个可以映射到用户空间的内存区域。对于每个区域，必须在 `mem[]` 数组中设置一个 `uio_mem` 结构体。以下是 `uio_mem` 结构体的字段描述：

- `const char *name`：可选的。设置此项可以帮助识别内存区域，它将显示在相应的 sysfs 节点中。
- `int memtype`：如果使用映射，则为必需项。如果您的卡上有要映射的物理内存，请将其设置为 `UIO_MEM_PHYS`。对于逻辑内存则使用 `UIO_MEM_LOGICAL`（例如，使用 **__get_free_pages()** 而不是 **kmalloc()** 进行分配的）。还有用于虚拟内存的 `UIO_MEM_VIRTUAL` 。
- `phys_addr_t addr`：如果使用映射，则为必需项。填写内存块的地址。这个地址就是出现在 sysfs 中的地址。
- `resource_size_t size`：填写 `addr` 所指向的内存块的大小。如果大小为零，则该映射被视为未使用。请注意，对于所有未使用的映射，必须将大小初始化为零。
- `void *internal_addr`：如果必须从内核模块中访问这个内存区域，则需要使用 **ioremap()** 等方法在内部映射它。此函数返回的地址无法映射到用户空间，因此不能将其存储在 `addr` 中。请改用 `internal_addr` 来记住这样的地址。

请不要触碰 `uio_mem` 结构体的 `map` 元素！UIO 框架使用它来设置此映射的 sysfs 文件。别管它。

有时，您的设备可能有一个或多个无法映射到用户空间的端口区域。但是，如果用户空间还有其他访问这些端口的可能性，那么在 sysfs 中提供有关端口的信息是有意义的。对于每个区域，您必须在 `port[]` 数组中设置一个 `uio_port` 结构体。以下是 `uio_port` 结构体字段的描述：

- `char *porttype`：必需的。将其设置为预定义的常量之一。对于 x86 体系结构中的 IO 端口，请使用 `UIO_PORT_X86` 。
- `unsigned long start`：如果使用端口区域，则为必需项。填写此区域的第一个端口的编号。
- `unsigned long size`：填写此区域中的端口数。如果大小为零，则认为该区域未使用。请注意，对于所有未使用的区域，必须将大小初始化为零。

请不要触碰 `uio_port` 结构体的 `portio` 元素！UIO 框架使用它来设置此映射的 sysfs 文件。别管它。

### 添加一个中断处理函数

您在中断处理程序中需要做的事情取决于您的硬件和您想要如何处理它。您应该尽量减少内核中断处理程序的代码量。如果您的硬件在每次中断后不需要执行任何操作，那么您的处理程序可以是空的。

另一方面，如果硬件在每次中断后都需要执行一些操作，那么必须在内核模块中执行。请注意，您不能依赖驱动程序的用户空间部分。您的用户空间程序可以随时终止，可能会使您的硬件处于仍然需要正确中断处理的状态。

也有一些应用场景，您可能希望在每次中断时从您的硬件读取数据，并将其缓存到您为此目的分配的一块内核内存中。通过这种技术，如果您的用户空间程序错过了一个中断，您可以避免数据丢失。

关于共享中断的注意事项：只要可能，您的驱动程序应该支持中断共享。这只有在您的驱动程序能够检测出是不是您的硬件触发了中断时才可行。这通常是通过查看中断状态寄存器来完成的。如果您的驱动程序看到 IRQ 位实际上被设置了，它将执行其操作，并且处理程序返回 IRQ_HANDLED 。如果驱动程序检测到不是您的硬件导致了中断，它将不执行任何操作并返回 IRQ_NONE，从而允许内核调用下一个可能的中断处理程序。

如果您决定不支持共享中断，您的卡将无法在没有空闲中断的计算机上工作。考虑到这种情况在PC平台上经常发生，您可以通过支持中断共享来避免很多麻烦。

### 将 uio_pdrv 用于平台设备

在许多情况下，平台设备的 UIO 驱动程序可以用通用的方式进行处理。在您定义 `platform_device` 结构体的同一个地方，只需实现中断处理程序并填充 `uio_info` 结构体。指向该 `uio_info` 结构体的指针将用作您平台设备的 `platform_data` 。

您还需要设置一个 `resource` 结构体数组，其中包含内存映射的地址和大小。此信息使用 `platform_device` 结构体的 `.resources` 和 `.num_resources` 元素传递给驱动程序。

现在，您必须将 `platform_device` 结构体的 `.name` 元素设置为 `"uio_pdrv"` 才能使用通用 UIO 平台设备驱动程序。该驱动程序将根据给定的资源填充 `mem[]`数组，并注册设备。

这种方法的优点是，您只需要编写一个无论如何都需要编写的文件。您不需要创建额外的驱动程序。

### 将 uio_pdrv_genirq 用于平台设备

尤其是在嵌入式设备中，您经常会发现 irq 引脚连接到自己的专用中断线的芯片。在这种情况下，如果您非常确定中断不是共享的，我们可以进一步了解 uio_pdrv 的概念，并使用通用中断处理程序。uio_pdrv_genirq 就是这么做的。

该驱动程序的设置与上面对 `uio_pdrv` 的描述相同，只是您没有实现中断处理程序。uio_info 结构体的 `.handler` 元素必须保持为 `NULL`。`.irq_flags` 元素不能包含 `IRQF_SHARED` 。

您将把 `platform_device` 结构体的 `.name` 元素设置为 `"uio_pdrv_genirq"` 以使用此驱动程序。

`uio_pdrv_genirq` 的通用中断处理程序将简单地使用 **disable_irq_nosync()** 禁用中断线。完成工作后，用户空间可以通过将 0x00000001 写入 UIO 设备文件来重新启用中断。驱动程序已经实现了 **irq_control()**，因此您不能实现自己的。

使用 `uio_pdrv_genirq` 不仅可以节省几行中断处理程序代码。您也不需要知道任何关于芯片内部寄存器的信息来创建驱动程序的内核部分。你只需要知道芯片所连接引脚的 irq 编号。

当在启用设备树的系统中使用时，驱动程序需要通过设置“of_id”模块参数为节点的“compatible”字符串来进行探测，以便处理该驱动程序应该管理的节点。默认情况下，节点的名称（没有单元地址）在用户空间中显示为 UIO 设备的名称。要设置自定义名称，可以在 DT 节点中指定一个名为 `"linux，uio-name"` 的属性。

### 将 uio_dmem_genirq 用于平台设备

除了静态分配的内存范围之外，它们还可能希望在用户空间驱动程序中使用动态分配的区域。特别是，能够访问通过 dma-mapping API 可用的内存可能特别有用。`uio_dmem_genirq` 驱动程序提供了一个方法来实现它。

在中断配置和处理方面，该驱动程序的使用方式与 `uio_pdrv_genirq` 驱动程序类似。

将 `platform_device` 结构体的 `.name` 元素设置为 `"uio_dmem_genirq"` 以使用此驱动程序。

使用此驱动程序时，请填写 `platform_device` 结构体的 `.platform_data` 元素，该元素的类型为`uio_dmem_genirq_pdata` 结构体，包含以下元素：

- `struct uio_info uioinfo`：与 `uio_pdrv_genirq` 平台数据使用的结构相同
- `unsigned int *dynamic_region_sizes`：指向要映射到用户空间的动态内存区域大小列表的指针。
- `unsigned int num_dynamic_regions`：`dynamic_regon_sizes` 数组中的元素数量。

平台数据中定义的动态区域将被附加到平台设备资源之后的 `mem[]` 数组中，这意味着静态和动态内存区域的总数不能超过 `MAX_UIO_MAPS` 。

当打开 UIO 设备文件 `/dev/uioX` 时，将分配动态内存区域。与静态内存资源类似，动态区域的内存区域信息也可以通过 sysfs 在 `/sys/class/uio/uioX/maps/mapY/*` 上看到。当 UIO 设备文件关闭时，动态内存区域将被释放。当没有进程打开设备文件时，返回给用户空间的地址为 ~0 。

## 在用户空间中编写驱动程序

一旦硬件有了可工作的内核模块，就可以编写驱动程序的用户空间部分。您不需要任何特殊的库，您的驱动程序可以用任何合理的语言编写，您可以使用浮点数等等。简而言之，您可以用您通常用来编写用户空间应用程序的所有工具和库。

### 获取 UIO 设备的信息

sysfs 中提供了有关所有 UIO 设备的信息。您应该在驱动程序中做的第一件事是检查 `name` 和 `version` ，以确保您使用的是正确的设备，并且其内核驱动程序具有您期望的版本。

您还应该确保您需要的内存映射存在并且具有您期望的大小。

有一个名为 `lsuio` 的工具可以列出 UIO 设备及其属性。在此处提供：

http://www.osadl.org/projects/downloads/UIO/user/

使用 `lsuio`，您可以快速检查内核模块是否已加载以及它导出了哪些属性。有关详细信息，请参阅手册页。

`lsuio` 的源代码可以作为获取 UIO 设备信息的示例。文件 `uio_helper.c` 包含许多可以在用户空间驱动程序代码中使用的函数。

### mmap() 设备内存

在您确定您拥有具有所需内存映射的正确设备后，您所要做的就是调用 **mmap()** 将设备的内存映射到用户空间。

**mmap()** 调用的参数 `offset` 对 UIO 设备有着特殊的意义：它用于选择要映射的设备映射。要映射第 N 个内存区域，您必须使用页面大小的 N 倍作为偏移量：

```
offset = N * getpagesize();
```

N 从零开始，所以如果只有一个内存范围要映射，请将 `offset` 设置为 0。这种技术的一个缺点是，内存总是从其起始地址开始映射。

### 等待中断

成功映射设备内存后，您可以像访问普通数组一样访问它。通常，您将执行一些初始化。之后，您的硬件开始工作，一旦完成，或有一些数据可用，或因为发生错误而需要您注意，就会立即产生中断。

`/dev/uioX` 是一个只读文件。在发生中断之前，**read()** 将始终处于阻塞状态。**read()** 的 `count` 参数只有一个合法值，即有符号的32位整数的大小（4）。`count` 的任何其他值都会导致 **read()** 失败。带符号的 32 位整数读取是设备的中断计数。如果该值比上次读取的值多 1，则一切正常。如果差值大于 1，则表明您错过了中断。

您也可以在 `/dev/uioX` 上使用 **select()** 。

## 通用 PCI UIO 驱动程序

通用驱动程序是一个名为 uio_pci_generic 的内核模块。它可以与任何符合 PCI 2.3（约2002年）的设备和任何符合 PCI Express 的设备一起工作。使用它，您只需要编写用户空间驱动程序，就不需要编写硬件特定的内核模块。

### 使驱动程序识别设备

由于驱动程序不声明任何设备 id，因此它不会自动加载，也不会自动绑定到任何设备，因此您必须自己加载它并将 id 分配给驱动程序。例如：

```
modprobe uio_pci_generic
echo "8086 10f5" > /sys/bus/pci/drivers/uio_pci_generic/new_id
```

如果您的设备已经有了特定于硬件的内核驱动程序，通用驱动程序仍然不会绑定到它，在这种情况下，如果您想使用通用驱动程序（为什么要？），您必须手动解除特定于硬件驱动程序的绑定，并绑定通用驱动程序，如下所示：

```
echo -n 0000:00:19.0 > /sys/bus/pci/drivers/e1000e/unbind
echo -n 0000:00:19.0 > /sys/bus/pci/drivers/uio_pci_generic/bind
```

您可以通过在 sysfs 中查找设备来验证该设备是否已绑定到驱动程序，例如如下所示：

```
ls -l /sys/bus/pci/devices/0000:00:19.0/driver
```

如果成功，则应该打印：

```
.../0000:00:19.0/driver -> ../../../bus/pci/drivers/uio_pci_generic
```

请注意，通用驱动程序不会绑定到旧的 PCI 2.2 设备。如果绑定设备失败，请运行以下命令：

```
dmesg
```

并在输出中查找错误原因。

### 关于 uio_pci_generic 的知识

中断是通过 PCI 命令寄存器中的中断禁用位和 PCI 状态寄存器中的中断状态位来处理的。所有符合 PCI 2.3（大约2002年）以及所有符合 PCI Express 标准的设备都应该支持这些位。uio_pci_generic 会检测到这种支持，并且不会绑定到不支持命令寄存器中的中断禁用位的设备。

在每次中断时，uio_pci_generic 都会设置中断禁用位。这可以防止设备在清除该位之前生成更多中断。用户空间驱动程序应该在阻塞和等待更多中断之前清除此位。

### 使用 uio_pci_generic 编写用户空间驱动程序

用户空间驱动程序可以使用 pci-sysfs 接口或封装它的 libpci 库与设备通信，并通过写入命令寄存器来重新启用中断。

### 使用 uio_pci_generic 的示例代码

```c
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <errno.h>

int main()
{
    int uiofd;
    int configfd;
    int err;
    int i;
    unsigned icount;
    unsigned char command_high;

    uiofd = open("/dev/uio0", O_RDONLY);
    if (uiofd < 0) {
        perror("uio open:");
        return errno;
    }
    configfd = open("/sys/class/uio/uio0/device/config", O_RDWR);
    if (configfd < 0) {
        perror("config open:");
        return errno;
    }

    /* Read and cache command value */
    err = pread(configfd, &command_high, 1, 5);
    if (err != 1) {
        perror("command config read:");
        return errno;
    }
    command_high &= ~0x4;

    for(i = 0;; ++i) {
        /* Print out a message, for debugging. */
        if (i == 0)
            fprintf(stderr, "Started uio test driver.\n");
        else
            fprintf(stderr, "Interrupts: %d\n", icount);

        /****************************************/
        /* Here we got an interrupt from the
           device. Do something to it. */
        /****************************************/

        /* Re-enable interrupts. */
        err = pwrite(configfd, &command_high, 1, 5);
        if (err != 1) {
            perror("config write:");
            break;
        }

        /* Wait for next interrupt. */
        err = read(uiofd, &icount, 4);
        if (err != 4) {
            perror("uio read:");
            break;
        }

    }
    return errno;
}
```

## 通用 Hyper-V UIO 驱动程序

通用驱动程序是一个名为 uio_hv_generic 的内核模块。它支持 Hyper-V 虚拟机总线上的设备，类似于 PCI 总线上的 uio_pci_generic 。

### 使驱动程序识别设备

由于驱动程序没有声明任何设备 GUID，因此它不会自动加载，也不会自动绑定到任何设备，因此您必须自己加载它并将 id 分配给驱动程序。例如，要使用网络设备类 GUID：

```
modprobe uio_hv_generic
echo "f8615163-df3e-46c5-913f-f2d2f965ed0e" > /sys/bus/vmbus/drivers/uio_hv_generic/new_id
```

如果设备已经有硬件特定的内核驱动程序，通用驱动程序仍然不会绑定到它，在这种情况下，如果您想为用户空间库使用通用驱动程序，则必须手动解除硬件特定驱动程序的绑定，并使用设备特定的 GUID 绑定通用驱动程序：

```
echo -n ed963694-e847-4b2a-85af-bc9cfc11d6f3 > /sys/bus/vmbus/drivers/hv_netvsc/unbind
echo -n ed963694-e847-4b2a-85af-bc9cfc11d6f3 > /sys/bus/vmbus/drivers/uio_hv_generic/bind
```

您可以通过在 sysfs 中查找设备来验证该设备是否已绑定到驱动程序，例如如下所示：

```
ls -l /sys/bus/vmbus/devices/ed963694-e847-4b2a-85af-bc9cfc11d6f3/driver
```

如果成功，则应该打印：

```
.../ed963694-e847-4b2a-85af-bc9cfc11d6f3/driver -> ../../../bus/vmbus/drivers/uio_hv_generic
```

### 关于 uio_hv_generic 的知识

在每次中断时，uio_hv_generic 都会设置中断禁用位。这可以防止设备在清除该位之前产生进一步的中断。用户空间驱动程序应该在阻塞和等待更多中断之前清除此位。

当主机撤销一个设备时，中断文件描述符会被标记为无效，任何对中断文件描述符的读取都将返回 -EIO 。这类似于一个关闭的套接字或断开连接的串行设备。

vmbus 设备区域被映射到 uio 设备资源中：

1. 通道环形缓冲区：从客户端到主机和从主机到客户端
2. 客户端到主机的中断信号页面
3. 客户端到主机的监控页面
4. 网络接收缓冲区
5. 网络发送缓冲区

如果通过向主机的请求创建了一个子通道，那么 uio_hv_generic 设备驱动程序将为每个通道的环形缓冲区创建一个 sysfs 二进制文件。例如：

```
/sys/bus/vmbus/devices/3811fe4d-0fa0-4b62-981a-74fc1084c757/channels/21/ring
```

## 更多信息

- [OSADL homepage](http://www.osadl.org/)
- [Linutronix homepage](http://www.linutronix.de/)
