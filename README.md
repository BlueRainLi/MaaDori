<!-- markdownlint-disable MD033 MD041 -->
<p align="center">
  <img alt="LOGO" src="https://cdn.jsdelivr.net/gh/MaaAssistantArknights/design@main/v1/icons/maa-logo_512x512.png" width="256" height="256" />
</p>

<div align="center">

# MaaDori

</div>

MaaDori 是基于 [MaaFramework](https://github.com/MaaXYZ/MaaFramework) 构建的自动化项目。

> **MaaFramework** 是基于图像识别技术、运用 [MAA](https://github.com/MaaAssistantArknights/MaaAssistantArknights) 开发经验去芜存菁、完全重写的新一代自动化黑盒测试框架。
> 低代码的同时仍拥有高扩展性，旨在打造一款丰富、领先、且实用的开源库，助力开发者轻松编写出更好的黑盒测试程序，并推广普及。

## 即刻开始

- [如何开发](./docs/zh_cn/develop/how_to_develop.md)
- [常见问题](./docs/zh_cn/develop/faq.md)
- [PR 规范](./docs/zh_cn/develop/pull_request_guidelines.md)

## 项目结构

```tree
MaaDori/
├── agent/               # 自定义识别 / 动作（Python）
├── assets/
│   └── MaaCommonAssets  # MaaFramework 通用资源子模块（OCR 模型等）
├── deps/                # MaaFramework 二进制与 JSON Schema
├── docs/                # 开发文档
├── resource/            # 流水线与图片资源（pipeline / image / model）
├── tools/               # 打包与校验脚本
├── check_resource.py    # 资源完整性检查
└── interface.json       # ProjectInterface V2 配置
```

## 反馈

发现问题或有功能建议，欢迎提交 [Issue](https://github.com/BlueRainLi/MaaDori/issues)。

## 鸣谢

本项目由 **[MaaFramework](https://github.com/MaaXYZ/MaaFramework)** 强力驱动！

[![Contributors](https://contrib.rocks/image?repo=MaaXYZ/MaaFramework&max=1000)](https://github.com/MaaXYZ/MaaFramework/graphs/contributors)
