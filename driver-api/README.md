> 原文：[Driver implementer's API guide — The Linux Kernel documentation](https://docs.kernel.org/driver-api/index.html)
>
> 译者：jklincn \<jklincn@outlook.com\>
>
> 日期：2024.03.19
>
> 修订：
>
> **已知问题：二级目录需要右键-在新标签页中打开链接才能正确跳转**

# 驱动程序开发者的 API 指南

内核提供了各种各样的接口来支持设备驱动程序的开发。本文档只是其中一些接口的一个有组织的集合——希望随着时间的推移，它会变得更好！可用的子部分如下所示。

## 设备驱动开发者须知文档

本节包含大多数从事设备驱动程序开发的开发人员在某些时候应该感兴趣的文档。

- [Driver Basics](https://docs.kernel.org/driver-api/basics.html)
- [Driver Model](https://docs.kernel.org/driver-api/driver-model/index.html)
- [Device links](https://docs.kernel.org/driver-api/device_link.html)
- [Device drivers infrastructure](https://docs.kernel.org/driver-api/infrastructure.html)
- [ioctl based interfaces](https://docs.kernel.org/driver-api/ioctl.html)
- [CPU and Device Power Management](https://docs.kernel.org/driver-api/pm/index.html)

## Useful support libraries[¶](https://docs.kernel.org/driver-api/index.html#useful-support-libraries)

This section contains documentation that should, at some point or other, be of interest to most developers working on device drivers.

- [Early Userspace](https://docs.kernel.org/driver-api/early-userspace/index.html)
- [Kernel Connector](https://docs.kernel.org/driver-api/connector.html)
- [Bus-Independent Device Accesses](https://docs.kernel.org/driver-api/device-io.html)
- [Device Frequency Scaling](https://docs.kernel.org/driver-api/devfreq.html)
- [Buffer Sharing and Synchronization (dma-buf)](https://docs.kernel.org/driver-api/dma-buf.html)
- [Component Helper for Aggregate Drivers](https://docs.kernel.org/driver-api/component.html)
- [The io_mapping functions](https://docs.kernel.org/driver-api/io-mapping.html)
- [Ordering I/O writes to memory-mapped addresses](https://docs.kernel.org/driver-api/io_ordering.html)
- [用户空间 I/O 指南](./uio-howto.md)
- [VFIO Mediated devices](https://docs.kernel.org/driver-api/vfio-mediated-device.html)
- [VFIO - “Virtual Function I/O”](https://docs.kernel.org/driver-api/vfio.html)
- [Acceptance criteria for vfio-pci device specific driver variants](https://docs.kernel.org/driver-api/vfio-pci-device-specific-driver-acceptance.html)

## Bus-level documentation

- [Auxiliary Bus](https://docs.kernel.org/driver-api/auxiliary_bus.html)
- [Compute Express Link](https://docs.kernel.org/driver-api/cxl/index.html)
- [EISA bus support](https://docs.kernel.org/driver-api/eisa.html)
- [Firewire (IEEE 1394) driver Interface Guide](https://docs.kernel.org/driver-api/firewire.html)
- [I3C subsystem](https://docs.kernel.org/driver-api/i3c/index.html)
- [ISA Drivers](https://docs.kernel.org/driver-api/isa.html)
- [MEN Chameleon Bus](https://docs.kernel.org/driver-api/men-chameleon-bus.html)
- [Linux PCI 驱动实现者的 API 指南](pci/README.md)
- [The Linux RapidIO Subsystem](https://docs.kernel.org/driver-api/rapidio/index.html)
- [Linux kernel SLIMbus support](https://docs.kernel.org/driver-api/slimbus.html)
- [Linux USB API](https://docs.kernel.org/driver-api/usb/index.html)
- [Virtio](virtio/README.md)
- [VME Device Drivers](https://docs.kernel.org/driver-api/vme.html)
- [W1: Dallas’ 1-wire bus](https://docs.kernel.org/driver-api/w1.html)
- [Xillybus driver for generic FPGA interface](https://docs.kernel.org/driver-api/xillybus.html)

## Subsystem-specific APIs

- [Linux 802.11 Driver Developer’s Guide](https://docs.kernel.org/driver-api/80211/index.html)
- [ACPI Support](https://docs.kernel.org/driver-api/acpi/index.html)
- [Kernel driver lp855x](https://docs.kernel.org/driver-api/backlight/lp855x-driver.html)
- [The Common Clk Framework](https://docs.kernel.org/driver-api/clk.html)
- [Console Drivers](https://docs.kernel.org/driver-api/console.html)
- [Crypto Drivers](https://docs.kernel.org/driver-api/crypto/index.html)
- [DMAEngine documentation](https://docs.kernel.org/driver-api/dmaengine/index.html)
- [The Linux kernel dpll subsystem](https://docs.kernel.org/driver-api/dpll.html)
- [Error Detection And Correction (EDAC) Devices](https://docs.kernel.org/driver-api/edac.html)
- [Linux Firmware API](https://docs.kernel.org/driver-api/firmware/index.html)
- [FPGA Subsystem](https://docs.kernel.org/driver-api/fpga/index.html)
- [Frame Buffer Library](https://docs.kernel.org/driver-api/frame-buffer.html)
- [Managing Ownership of the Framebuffer Aperture](https://docs.kernel.org/driver-api/aperture.html)
- [Generic Counter Interface](https://docs.kernel.org/driver-api/generic-counter.html)
- [General Purpose Input/Output (GPIO)](https://docs.kernel.org/driver-api/gpio/index.html)
- [High Speed Synchronous Serial Interface (HSI)](https://docs.kernel.org/driver-api/hsi.html)
- [The Linux Hardware Timestamping Engine (HTE)](https://docs.kernel.org/driver-api/hte/index.html)
- [I2C and SMBus Subsystem](https://docs.kernel.org/driver-api/i2c.html)
- [Industrial I/O](https://docs.kernel.org/driver-api/iio/index.html)
- [InfiniBand and Remote DMA (RDMA) Interfaces](https://docs.kernel.org/driver-api/infiniband.html)
- [Input Subsystem](https://docs.kernel.org/driver-api/input.html)
- [Generic System Interconnect Subsystem](https://docs.kernel.org/driver-api/interconnect.html)
- [IPMB Driver for a Satellite MC](https://docs.kernel.org/driver-api/ipmb.html)
- [The Linux IPMI Driver](https://docs.kernel.org/driver-api/ipmi.html)
- [libATA Developer’s Guide](https://docs.kernel.org/driver-api/libata.html)
- [The Common Mailbox Framework](https://docs.kernel.org/driver-api/mailbox.html)
- [RAID](https://docs.kernel.org/driver-api/md/index.html)
- [Media subsystem kernel internal API](https://docs.kernel.org/driver-api/media/index.html)
- [Intel(R) Management Engine Interface (Intel(R) MEI)](https://docs.kernel.org/driver-api/mei/index.html)
- [Memory Controller drivers](https://docs.kernel.org/driver-api/memory-devices/index.html)
- [Message-based devices](https://docs.kernel.org/driver-api/message-based.html)
- [Miscellaneous Devices](https://docs.kernel.org/driver-api/misc_devices.html)
- [Parallel Port Devices](https://docs.kernel.org/driver-api/miscellaneous.html)
- [16x50 UART Driver](https://docs.kernel.org/driver-api/miscellaneous.html#x50-uart-driver)
- [Pulse-Width Modulation (PWM)](https://docs.kernel.org/driver-api/miscellaneous.html#pulse-width-modulation-pwm)
- [MMC/SD/SDIO card support](https://docs.kernel.org/driver-api/mmc/index.html)
- [Memory Technology Device (MTD)](https://docs.kernel.org/driver-api/mtd/index.html)
- [MTD NAND Driver Programming Interface](https://docs.kernel.org/driver-api/mtdnand.html)
- [Near Field Communication](https://docs.kernel.org/driver-api/nfc/index.html)
- [NTB 驱动程序](ntb.md)
- [Non-Volatile Memory Device (NVDIMM)](https://docs.kernel.org/driver-api/nvdimm/index.html)
- [NVMEM Subsystem](https://docs.kernel.org/driver-api/nvmem.html)
- [PARPORT interface documentation](https://docs.kernel.org/driver-api/parport-lowlevel.html)
- [Generic PHY Framework](https://docs.kernel.org/driver-api/phy/index.html)
- [PINCTRL (PIN CONTROL) subsystem](https://docs.kernel.org/driver-api/pin-control.html)
- [PLDM Firmware Flash Update Library](https://docs.kernel.org/driver-api/pldmfw/index.html)
- [Overview of the `pldmfw` library](https://docs.kernel.org/driver-api/pldmfw/index.html#overview-of-the-pldmfw-library)
- [PPS - Pulse Per Second](https://docs.kernel.org/driver-api/pps.html)
- [PTP hardware clock infrastructure for Linux](https://docs.kernel.org/driver-api/ptp.html)
- [Pulse Width Modulation (PWM) interface](https://docs.kernel.org/driver-api/pwm.html)
- [Voltage and current regulator API](https://docs.kernel.org/driver-api/regulator.html)
- [Reset controller API](https://docs.kernel.org/driver-api/reset.html)
- [rfkill - RF kill switch support](https://docs.kernel.org/driver-api/rfkill.html)
- [Writing s390 channel device drivers](https://docs.kernel.org/driver-api/s390-drivers.html)
- [SCSI Interfaces Guide](https://docs.kernel.org/driver-api/scsi.html)
- [Support for Serial devices](https://docs.kernel.org/driver-api/serial/index.html)
- [SM501 Driver](https://docs.kernel.org/driver-api/sm501.html)
- [SoundWire Documentation](https://docs.kernel.org/driver-api/soundwire/index.html)
- [Serial Peripheral Interface (SPI)](https://docs.kernel.org/driver-api/spi.html)
- [Surface System Aggregator Module (SSAM)](https://docs.kernel.org/driver-api/surface_aggregator/index.html)
- [Linux Switchtec Support](https://docs.kernel.org/driver-api/switchtec.html)
- [Sync File API Guide](https://docs.kernel.org/driver-api/sync_file.html)
- [target and iSCSI Interfaces Guide](https://docs.kernel.org/driver-api/target.html)
- [TEE (Trusted Execution Environment) driver API](https://docs.kernel.org/driver-api/tee.html)
- [Thermal](https://docs.kernel.org/driver-api/thermal/index.html)
- [TTY](https://docs.kernel.org/driver-api/tty/index.html)
- [WBRF - Wifi Band RFI Mitigations](https://docs.kernel.org/driver-api/wbrf.html)
- [WMI Driver API](https://docs.kernel.org/driver-api/wmi.html)
- [Xilinx FPGA](https://docs.kernel.org/driver-api/xilinx/index.html)
- [Writing Device Drivers for Zorro Devices](https://docs.kernel.org/driver-api/zorro.html)
