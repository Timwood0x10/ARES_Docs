---
title: "模块"
description: "每个 ARES 模块的包级参考，基于真实源码扫描生成。"
weight: 3
---

下方每个模块页面均基于对应 Go 包的真实源码扫描生成。每页包含职责摘要、
Mermaid 架构图、导出接口与关键方法、模块协作关系、具体的扩展方式，以及
成熟度标注（Production / Beta / Experimental）。

成熟度标准：

- **Production** — 有测试覆盖、无实验性标记、已集成进 SDK 入口。
- **Beta** — 功能完备且有测试，但公开 API 仍在演进。
- **Experimental** — 新模块或正在大幅重构，API 可能变更。
