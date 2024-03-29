# Linux 内核文档中文翻译

![Dynamic TOML Badge](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fgist.githubusercontent.com%2Fjklincn%2Ffeda703c740af0973eed518fbb50c1bb%2Fraw&query=%24.count&style=flat-square&label=%E5%AD%97%E7%AC%A6%E7%BB%9F%E8%AE%A1)![Dynamic TOML Badge](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fgist.githubusercontent.com%2Fjklincn%2Ffeda703c740af0973eed518fbb50c1bb%2Fraw%2F&query=%24.article&style=flat-square&label=%E6%96%87%E7%AB%A0%E6%95%B0%E9%87%8F)

Linux 内核文档有非常多高质量的文档，但国内的翻译项目貌似都已经停止：

1. https://docs.kernel.org/translations/zh_CN/index.html
2. https://tinylab-1.gitbook.io/linux-doc/zh-cn

最近在读 Linux 内核文档，想着翻译一些，加深自己理解的同时也为后来的人铺一条路，于是就有了这个仓库。

个人能力有限，欢迎一起补充。

# 目录

1. [核心 API 文档](core-api/README.md)
1. [PCI 总线子系统](pci/README.md)
1. [驱动程序开发者的 API 指南](driver-api/README.md)
1. [Linux 跟踪技术](trace/README.md)

# 注意事项

- 文件路径严格按照官方文档设置，字母均为小写。`index.html` 修改为 `README.md`，便于在 GitHub 中浏览

  例如：`https://docs.kernel.org/driver-api/pci/p2pdma.html` 在仓库中的路径为 `driver-api/pci/p2pdma.md`

- GitHub中锚点的生成规则为：

  - 英文大写转换为小写

  - 空格替换为短划线 `-`
  - 省略标题中的 `.`  `(`   `)` 符号（中文括号不省略）

  例如：标题 `## 1.2. pci_register_driver() 调用` 转换为 `#12-pci_register_driver-调用`

- Commit 信息规范：\<type\>: <filename\>, ..., <filename\>

  其中 \<type\> 可以为以下类型，filename 为完整的相对路径，去除 md 后缀

  - add：添加新文档
  - update：更新文档内容
  - style：仅做格式的修改
  - organize：整理仓库/对文档进行分类
