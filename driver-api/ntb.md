> 原文：[NTB Drivers — The Linux Kernel documentation](https://docs.kernel.org/driver-api/ntb.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.15
>
> 修订：

# NTB 驱动程序

NTB（Non-Transparent Bridge，非透明桥）是一种 PCI-Express 桥接芯片类型，它将两台或更多计算机的独立内存系统连接到同一个 PCI-Express 结构上。现有的 NTB 硬件支持一个通用功能集：门铃（原文：doorbell）寄存器和内存转换窗口，以及非通用功能，如便笺（原文：scratchpad）寄存器和消息寄存器。便笺寄存器是可从设备任意一端访问的读写寄存器，因此对等方可以在一个固定的地址交换少量信息。消息寄存器可以用于相同的目的。此外，它们还提供了特殊的状态位，以确保信息不会被另一个对等方重写。门铃寄存器为对等方提供了一种发送中断事件的方式。内存窗口允许对对等方的内存进行转换后的读写访问。

## NTB 核心驱动程序（ntb）

NTB 核心驱动定义了一个封装通用功能集的 API，并允许对 NTB 功能感兴趣的客户端发现硬件驱动支持的 NTB 设备。这里的“客户端”是指调用 NTB API 的上层组件。而“驱动”或“硬件驱动”是指针对特定厂商和型号的 NTB 硬件驱动程序。

## NTB 客户端驱动程序

NTB 客户端驱动应当向 NTB 核心驱动注册。注册后，当 NTB 硬件或硬件驱动插入和移除时，客户端的探测和移除函数将会适当地被调用。注册使用的是 Linux 设备框架，因此对于任何编写过 PCI 驱动的人来说应该感觉很熟悉。

### NTB 典型客户端驱动程序实现

NTB的主要目的是在至少两个系统之间共享一部分内存。因此，像便笺/消息寄存器这样的NTB设备功能主要用于执行正确的内存窗口初始化。通常，NTB API支持两种类型的内存窗口接口：在本地 NTB 端口上配置的入站转换和在对等 NTB 端口由对等方配置的出站转换。第一种类型如下图所示：

```
    Inbound translation:

    Memory:              Local NTB Port:      Peer NTB Port:      Peer MMIO:
     ____________
    | dma-mapped |-ntb_mw_set_trans(addr)  |
    | memory     |        _v____________   |   ______________
    | (addr)     |<======| MW xlat addr |<====| MW base addr |<== memory-mapped IO
    |------------|       |--------------|  |  |--------------|
```

因此，第一种类型的内存窗口初始化的典型场景是：1) 分配一个内存区域，2) 将转换后的地址放入 NTB 配置中，3) 以某种方式通知对等设备已进行初始化，4) 对等设备映射相应的出站内存窗口，从而可以访问共享内存区域。

第二种类型的接口，即由对端设备初始化共享窗口的情况，描绘如下：

```
    Outbound translation:

    Memory:        Local NTB Port:    Peer NTB Port:      Peer MMIO:
     ____________                      ______________
    | dma-mapped |                |   | MW base addr |<== memory-mapped IO
    | memory     |                |   |--------------|
    | (addr)     |<===================| MW xlat addr |<-ntb_peer_mw_set_trans(addr)
    |------------|                |   |--------------|
```

第二种类型的接口初始化的典型场景是：1）分配一个内存区域，2）以某种方式向对等设备发送转换后的地址，3）对等设备将转换后的地址放入 NTB 配置中，4）对等设备映射出站内存窗口，从而可以访问共享内存区域。

可以看出，所描述的场景可以组合在一个可移植的算法中。

- 本地设备：
  		1. 为共享窗口分配内存
    		1. 使用已分配区域的转换后的地址初始化内存窗口（如果不支持本地内存窗口初始化，此操作可能会失败）
    		1. 将转换后的地址和内存窗口索引发送给对等设备
- 对等设备：
  1. 使用检索到的由另一个设备分配的内存区域地址来初始化内存窗口（如果不支持对等内存窗口初始化，此操作可能会失败）
  2. 映射出站内存窗口

根据这个场景，NTB 内存窗口 API 可以如下使用：

> 译者注：下文提及的 pidx 指端口索引（port index），midx 指内存窗口索引（memory window index）

- 本地设备：
  1. ntb_mw_count(pidx) - 获取内存区域的数量，这些区域是可以用来分配本地设备和指定索引端口的对等设备之间的内存窗口。
  2. ntb_get_align(pidx, midx) - 获取限制共享内存区域对齐和大小的参数。然后就可以正确地分配内存。
  3. 根据第 2 步获取到的限制分配物理上连续的内存区域。
  4. ntb_mw_set_trans(pidx, midx) - 尝试为定义的对等设备设置指定索引的内存窗口的转换地址（如果不支持设置本地转换地址，此操作可能会失败）
  5. 使用便笺寄存器或消息寄存器等方式，将转换后的基址（通常与内存窗口索引一起）发送给对等设备。
- 对等设备：
  1. ntb_peer_mw_set_trans(pidx, midx) - 尝试为指定内存窗口设置从其他设备（与 pidx 相关）接收到的转换地址。如果接收的地址超过了最大可能的地址或没有正确对齐，此操作可能会失败。
  2. ntb_peer_mw_get_addr(widx) - 获取 MMIO 地址以映射内存窗口，从而可以访问共享内存。

另外值得注意的是，方法 ntb_mw_count(pidx) 应该与对等方端口索引为 pidx 的 ntb_peer_mw_count() 返回相同的值。

### NTB 传输客户端（ntb_transport）和 NTB 网络设备驱动（ntb_netdev）

NTB 的主要客户端是传输客户端，与 NTB 网络设备驱动配合使用。这些驱动一起工作，通过 NTB 创建一个到对等方的逻辑链路，以交换网络数据包。传输客户端建立到对等方的逻辑链接，并创建队列对来交换消息和数据。然后 NTB 网络设备驱动使用传输队列对创建一个以太网设备。网络数据在套接字缓冲区和传输队列对缓冲区之间复制。传输客户端除了用于网络设备驱动外，还可以用于其他用途，不过目前尚未编写其他应用程序。

### NTB Ping Pong 测试客户端（ntb\_pingpong）

Ping Pong 测试客户端可以演示如何使用 NTB 硬件的门铃寄存器和便笺寄存器，同时也是一个简单的 NTB 客户端示例。Ping Pong 在启动时启用链接，等待 NTB 链接建立后，开始读写 NTB 的门铃寄存器和便笺寄存器。对等方使用门铃寄存器的位掩码互相中断，每一轮将其向左移动一位，以测试多个门铃寄存器的位和中断向量的行为。Ping Pong 驱动程序还会在每一轮写对等方的门铃寄存器之前，读取本地的第一个便笺寄存器，将值加 1 后写入对等方的第一个便笺寄存器。

模块参数：

- unsafe - 一些硬件在便笺寄存器和门铃寄存器上存在已知问题。默认情况下，Ping Pong 不会尝试操作此类硬件。您可以通过设置 unsafe=1 来覆盖此行为，但风险由您自己承担。
- delay_ms - 指定接收门铃中断事件和为下一轮设置对等方的门铃寄存器之间的延迟。
- init_db - 指定门铃寄存器的位以开始新的轮次。一旦所有门铃寄存器的位都被移出范围，一个新的轮次就开始了。
- dyndbg - 建议在加载此模块时指定 dyndbg=+p，然后在控制台上观察调试输出。

### NTB 工具测试客户端（ntb_tool）

工具测试客户端主要用于调试 NTB 硬件和驱动程序。该工具通过 debugfs 提供访问权限，用于读取、设置和清除 NTB 门铃寄存器，以及读写便笺寄存器。

该工具目前没有任何模块参数。

Debugfs 文件：

- *debugfs*/ntb_tool/*hw*/

  将为工具探测到的每个NTB设备在 debugfs 中创建一个目录。以下将这个目录简称为 *hw* 。

- *hw*/db

  该文件用于读取、设置和清除本地门铃寄存器。并非所有操作都被硬件支持。如果要读取门铃寄存器，可以读取该文件。如果要设置门铃寄存器，先写 s，再写要设置的值（例如：*echo 's 0x0101' > db*）。如果要清除门铃寄存器，先写 c，再写要清除的位。

- *hw*/mask

  该文件用于读取、设置和清除本地门铃寄存器掩码。详情见 *db* 。

- *hw*/peer_db

  该文件用于读取、设置和清除对等方的门铃寄存器。详情见 *db* 。

- *hw*/peer_mask

  该文件用于读取、设置和清除对等方的门铃寄存器掩码。详情见 *db* 。

- *hw*/spad

  该文件用于读取和写入本地便笺寄存器。如果要读取所有便笺寄存器的值，可以读取该文件。如果要写入值，请写入一系列便笺寄存器编号和值组成的对（例如：*echo '4 0x123 7 0xabc' > spad* # 分别设置 4 号便笺寄存器和 7 号便笺寄存器的值为 0x123 和 0xabc ）。

- *hw*/peer_spad

  该文件用于读取和写入对等方的便笺寄存器。详情见 *spad* 。

### NTB MSI 测试客户端（ntb_msi_test）

MSI 测试客户端用于测试和调试 MSI 库，该库允许通过 NTB 内存窗口发送 MSI 中断。测试客户端通过 debugfs 文件系统进行交互：

- *debugfs*/ntb_msi_test/*hw*/

  将为 msi 测试探测到的每个NTB设备在debugfs中创建一个目录。以下将这个目录简称为 *hw* 。

- *hw*/port

  该文件描述了本地端口号。

- *hw*/irq*_occurrences

  每个中断都有一个事件文件，读取时会返回触发中断的次数。

- *hw*/peer*/port

  该文件描述了每个对等方的端口号。

- *hw*/peer*/count

  该文件描述了每个对等方可以触发的中断数量

- *hw*/peer*/trigger

  写入一个中断号（任何小于 count 中指定值的数字）将触发指定对等方上的中断。该对等方的中断事件文件应该递增。

## NTB 硬件驱动程序

NTB 硬件驱动程序应使用 NTB 核心驱动程序注册设备。注册后，将调用客户端的探测和删除函数。

### NTB Intel 硬件驱动（ntb_hw_intel）

Intel 硬件驱动程序支持在 Xeon 和 Atom CPU 上使用 NTB 。

模块参数：

- b2b_mw_idx

  如果要通过内存窗口访问对等方的 NTB，则使用此内存窗口访问该对等方的 NTB。零或正值从第一个 mw idx 开始，负值从最后一个 mw idex 开始。双方必须在此处设置相同的值！默认值为 -1 。

- b2b_mw_share

  如果要通过内存窗口访问对等方的 NTB，并且如果内存窗口足够大，则仍然允许客户端使用内存窗口的后半部分来向对等方进行地址转换。

- xeon_b2b_usd_bar2_addr64

  如果在 Xeon 硬件上使用 B2B 拓扑，则在链路上游侧的 BAR2 窗口的 NTB 设备之间的总线上使用此 64 位地址。

- xeon_b2b_usd_bar4_addr64 - 见 *xeon_b2b_bar2_addr64* 。

- xeon_b2b_usd_bar4_addr32 - 见 *xeon_b2b_bar2_addr64* 。

- xeon_b2b_usd_bar5_addr32 - 见 *xeon_b2b_bar2_addr64* 。

- xeon_b2b_dsd_bar2_addr64 - 见 *xeon_b2b_bar2_addr64* 。

- xeon_b2b_dsd_bar4_addr64 - 见 *xeon_b2b_bar2_addr64* 。

- xeon_b2b_dsd_bar4_addr32 - 见 *xeon_b2b_bar2_addr64* 。

- xeon_b2b_dsd_bar5_addr32 - 见 *xeon_b2b_bar2_addr64* 。
