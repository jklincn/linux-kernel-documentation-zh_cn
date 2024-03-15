# Linux 内核文档中文翻译

Linux 内核文档有非常多高质量的文档，但国内的翻译项目貌似都已经停止：

1. https://docs.kernel.org/translations/zh_CN/index.html
2. https://tinylab-1.gitbook.io/linux-doc/zh-cn

最近在读 Linux 内核文档，想着翻译一些，加深自己理解的同时也为后来的人铺一条路，于是就有了这个仓库。

个人能力有限，欢迎一起补充。

# 目录

**数量有限则暂时不进行分类**

1. [PCI 总线子系统](pci_index.md)
1. [NTB 驱动程序](driver-api_ntb.md)

# 注意事项

- markdown 文件名根据文档相应的 URL 命名，字母均为小写。

  例如，https://docs.kernel.org/PCI/index.html 的翻译文档命名为 pci_index.md 。

- 原文中的表格可以按照 markdown 表格语法制作，但表头设为空行。

- GitHub中锚点的生成规则为：

  - 英文大写转换为小写

  - 空格替换为短划线 `-`
  - 省略标题中的 `.`  `(`   `)` 符号

  例如：标题 `## 1.2. pci_register_driver() 调用` 转换为 `#12-pci_register_driver-调用`
