---
title: "扩展指南"
description: "如何为 ARES 扩展自定义 LLM 供应商、工具、知识存储与策略源。"
weight: 2
---

ARES 在每一层都设计了扩展点。以下各节引用了需要实现的确切 SDK 选项或接口。

## 新增 LLM 供应商

SDK 内置支持 OpenAI、Ollama、Anthropic 与 OpenRouter。使用自定义供应商：

1. 实现 `llm.Client` 接口（`Generate`、`Chat`、`GenerateEmbedding`、
   `GenerateStream`）。
2. 构造 `*core.LLMConfig`，填入供应商名称与 base URL。
3. 通过 `sdk.WithLLMConfig(cfg)` 传入，或用 `sdk.WithFallbackLLM(cfg)`
   添加故障转移。

## 新增自定义工具

1. 实现 `tools.Tool` 接口（`Name`、`Description`、`Execute`、`Parameters`、
   `Capabilities`），或用 `tools.ToolFunc` 包装函数。
2. 注册：`runtime.RegisterTool(myTool)`，或作为 `sdk.WithTool` agent 选项传入。
3. 若需在 sub-agent 中注册幂等（可安全重试）工具，将 `ToolBinder` 类型断言
   为 `*toolBinder` 并调用 `BindIdempotentTool`。

## 新增知识存储后端

1. 实现 `knowledge.KnowledgeStore` 接口（13 个方法，含 `Store`、`Query`、
   `HybridSearch`、`FindDuplicate`）。
2. 通过 `sdk.WithKnowledgeStore(myStore)` 接入，或作为
   `provider.GraphProvider` 用 `sdk.WithKnowledgeProvider` 注册。

## 新增策略源

1. 实现 `agents.StrategySource`，返回当前 `ActiveStrategy`（prompt + LLM 参数）。
2. 构造 agent 时通过 `leader.WithStrategySource(src)` 或
   `sub.WithStrategySource(src)` 注入。

## 新增 MCP 服务器连接

1. 启动一个通过 stdio 暴露工具的 MCP 服务器。
2. 用 `sdk.WithMCP(MCPConn{Command: "...", Args: []string{...}})` 连接。
   发现的工具会自动注册到工具注册表。

## 调整知识质量门

使用 `sdk.WithAKGQualityGate(knowledge.QualityGateConfig{...})` 调整
抽取/一致性/新鲜度/使用度权重与分数阈值，决定哪些候选成为活跃的
KnowledgeObject。
