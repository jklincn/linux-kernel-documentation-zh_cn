> 原文：[3. PCI Express I/O Virtualization Howto  — The Linux Kernel documentation](https://docs.kernel.org/PCI/pci-iov-howto.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.18
>
> 修订：

# 3. PCI Express I/O 虚拟化指南

## 3.1. 概述

### 3.1.1. 什么是 SR-IOV

单根 I/O 虚拟化（SR-IOV）是一种 PCI Express 扩展功能，它使一个物理设备看起来像多个虚拟设备。物理设备称为物理功能（PF），而虚拟设备称为虚拟功能（VF）。VF 的分配可以通过封装在功能中的寄存器受 PF 动态控制。默认情况下，此功能未启用，PF 的行为与传统 PCIe 设备相同。一旦开启，每个 VF 的 PCI 配置空间都可以通过其自己的总线、设备和功能编号（路由 ID）访问。每个 VF 也有 PCI 内存空间，用于映射其寄存器集。VF 设备驱动程序对寄存器集进行操作，因此它可以正常工作，并显示为真实的现有 PCI 设备。

## 3.2. 用户指南

### 3.2.1. 如何启用 SR-IOV 功能

有多种方法可用于 SR-IOV 启用。在第一种方法中，设备驱动程序（PF 驱动程序）将通过 SR-IOV 核心提供的 API 来控制功能的启用和禁用。如果硬件具有 SR-IOV 功能，加载其 PF 驱动程序将启用它以及与 PF 相关的所有 VF 。一些 PF 驱动程序需要设置模块参数来确定要启用的 VF 数量。在第二种方法中，对 sysfs 文件 sriov_numvfs 的写入将启用和禁用与 PCIe PF 相关联的 VF。此方法启用每个 PF 的 VF 启用/禁用值，而第一种方法会启用同一设备的所有 PF。此外，PCI SRIOV 核心支持确保启用/禁用操作有效，以减少多个驱动程序中的相同检查，例如，如果启用VF，会检查 numvfs == 0，并确保 numvfs <= totalvfs。第二种方法是新的/未来的 VF 设备的推荐方法。

### 3.2.2. 如何使用虚拟功能

VF 在内核中被视为热插拔 PCI 设备，因此它们应该能够以与真实PCI设备相同的方式工作。VF 和普通 PCI 设备一样也需要设备驱动程序。

## 3.3. 开发者指南

### 3.3.1. SR-IOV API

要启用 SR-IOV 功能：

1. 对于第一种方法，在驱动程序中：

   ```c
   int pci_enable_sriov(struct pci_dev *dev, int nr_virtfn);
   ```

   “nr_virtfn” 是要启用的 VF 数量。

2. 对于第二种方法，从 sysfs：

   ```shell
   echo 'nr_virtfn' > \
   /sys/bus/pci/devices/<DOMAIN:BUS:DEVICE.FUNCTION>/sriov_numvfs
   ```

要禁用 SR-IOV 功能：

1. 对于第一种方法，在驱动程序中：

   ```c
   void pci_disable_sriov(struct pci_dev *dev);
   ```

2. 对于第二种方法，从 sysfs：

   ```shell
   echo  0 > \
   /sys/bus/pci/devices/<DOMAIN:BUS:DEVICE.FUNCTION>/sriov_numvfs
   ```

要通过主机上的兼容驱动程序启用自动探测 VF，请在启用 SR-IOV 功能之前运行以下命令。这是默认行为。

```shell
echo 1 > \
/sys/bus/pci/devices/<DOMAIN:BUS:DEVICE.FUNCTION>/sriov_drivers_autoprobe
```

要通过主机上的兼容驱动程序禁用自动探测 VF，请在启用 SR-IOV 功能之前运行以下命令。更新此条目不会影响已探测的 VF 。

```shell
echo  0 > \
/sys/bus/pci/devices/<DOMAIN:BUS:DEVICE.FUNCTION>/sriov_drivers_autoprobe
```

### 3.3.2. 使用示例

下面的代码说明了 SR-IOV API 的使用。

```c
static int dev_probe(struct pci_dev *dev, const struct pci_device_id *id)
{
        pci_enable_sriov(dev, NR_VIRTFN);

        ...

        return 0;
}

static void dev_remove(struct pci_dev *dev)
{
        pci_disable_sriov(dev);

        ...
}

static int dev_suspend(struct device *dev)
{
        ...

        return 0;
}

static int dev_resume(struct device *dev)
{
        ...

        return 0;
}

static void dev_shutdown(struct pci_dev *dev)
{
        ...
}

static int dev_sriov_configure(struct pci_dev *dev, int numvfs)
{
        if (numvfs > 0) {
                ...
                pci_enable_sriov(dev, numvfs);
                ...
                return numvfs;
        }
        if (numvfs == 0) {
                ....
                pci_disable_sriov(dev);
                ...
                return 0;
        }
}

static struct pci_driver dev_driver = {
        .name =         "SR-IOV Physical Function driver",
        .id_table =     dev_id_table,
        .probe =        dev_probe,
        .remove =       dev_remove,
        .driver.pm =    &dev_pm_ops,
        .shutdown =     dev_shutdown,
        .sriov_configure = dev_sriov_configure,
};
```

